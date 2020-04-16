# Cisco FlexVPN remote access server with FreeRADIUS and OpenLDAP
We recently got a [Cisco ISR 4451-X](https://www.cisco.com/c/en/us/support/routers/4451-x-integrated-services-router-isr/model.html),
which supports the newest Cisco's VPN solution, **FlexVPN**, based on IKEv2/IPsec.
This potentially allows clients to use any IKEv2-compatible client, e.g. [strongSwan](https://www.strongswan.org/) on Linux or native VPN client for Windows/macOS. However,
we found little documentation on the integration of a FlexVPN server with an existing [OpenLDAP](https://www.openldap.org/)-based authentication infrastructure, using a
[FreeRADIUS](https://freeradius.org/) server to allow EAP authentication. Most of documentation found online either used
Cisco's proprietary client (AnyConnect), an Active Directory-based authentication infrastructure, or certificate-based authentication.
For this reason, I share here the configuration that we applied, which is far to be optimal. If you have any suggestion, feel free
to send an e-mail!

## General overview
The goal is setting up a standard remote-access VPN. We want that road-warriors (i.e. remote-access clients) access the internal network. Since we also want
that remote access users access the Internet using the VPN, we do not configure the so-called *split-tunneling*. However,
this can be achieved by applying an appropriate client configuration, keeping in mind that not all clients fully supports it.

For this guide, we will assume the following network settings:
 * Router public IP address: `1.1.1.1`
 * Router name: `rt.example.com`
 * Router private IP address: `10.0.0.1/24`
 * AAA server IP address: `10.0.0.2`
 * AAA server name: `aaa.example.com`

## AAA server configuration
The AAA server provides authentication and authorization for the FlexVPN server using EAP. Optionally, it can also provide
accounting, but this is left for future investigation.

The AAA server uses the following software to provide EAP authentication:
 * [OpenLDAP](https://www.openldap.org/) as user database.
 * [FreeRADIUS](https://freeradius.org/) for EAP AAA.
 As Operating System, any modern GNU/Linux distribution is sufficient. For this guide,
 I will use [Debian](https://www.debian.org/) Linux.

### LDAP database
The first step is the configuration of the OpenLDAP database, which contains the list
of users which are authorized to access the VPN.

The first choice is the schema to adopt. Potentially, any LDAP schema which contains users and passwords is fine. However, for optimal
results, it is best to use FreeRADIUS's [schema](https://github.com/FreeRADIUS/freeradius-server/blob/master/doc/schemas/ldap/openldap/freeradius.ldif).
FreeRADIUS can be configured to use multiple attributes. For this guide, only the following attributes will be used:
 * **uid** as username.
 * **userPassword** as SSHA-hashed password for PAP authentication.
 * **sambaNTPassword** as NT-Password, used for MSCHAPv2 authentication.
 * **radiusFramedIPAddress** as static IP address pushed to the clients (optional).

For this reason, the users will have the *inetOrgPerson* structural schema, along with the *sambaSamAccount* and *radiusprofile* auxiliary schemas.
Since *sambaSamAccount* requires an attribute *sambaSID*, a random one is generated.

#### Example user
Assuming that all the users are in the organizational unit *Users*, an example user has the following entry in the LDAP database, in LDIF format:
```
dn: uid=n.surname,ou=Users,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: radiusprofile
objectClass: sambaSamAccount
objectClass: top
objectClass: uidObject
cn: Name Surname
sambaSID: S-1-5-21-1104974656-1107154463-2682333131
sn: Surname
uid: n.surname
displayName: Name Surname
givenName: Name
mail: n.surname@example.com
preferredLanguage: en
radiusFramedIPAddress: 10.0.0.100
sambaNTPassword: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
userPassword:: {SSHA}XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### RADIUS server
As RADIUS server, we used *FreeRADIUS*. Unfortunately, *FreeRADIUS* documentation on the website is very poor, making very difficult to build a working configuration from scratch.
Therefore, we started from a working FreeRADIUS configuration for *eduroam*, taken from the [Italian documentation](https://docs.eduroam.it/configurazione/freeradius.html). Then,
we modified the configuration as follows.
 - We removed all lines in the configuration files regarding the Chargeable User Identity (CUI) and the Operator-Name, since they are not used.
 - We removed the national eduroam client and proxy servers.
 - We used the same TLS certificate, generated by a well-known CA, used by the Cisco router for IKEv2 identification (**rt.example.com**).
 - We disabled EAP-GTC authentication, and enabled EAP-MSCHAPv2 authentication. Therefore, the server allows authentication with the following EAP methods:
   - EAP-MSCHAPv2
   - EAP-TTLS with PAP
   - EAP-PEAP with MSCHAPv2
In principle, all clients support EAP-MSCHAPv2 authentication. However, we prefer tunneled EAP authentication to protect user passwords from compromises of the router and/or the internal network between router and RADIUS server.

The RADIUS configuration files that were more heavily modified from eduroam's configurations are here reported.
 - `/etc/freeradius/sites-enabled/vpn`:
```
server vpn {

	listen {
		type = "auth"
		ipaddr = *
		port = 0
	}
	listen {
		type = "acct"
		ipaddr = *
		port = 0
	}
	listen {
		type = "auth"
		ipv6addr = ::
		port = 0
	}
	listen {
		type = "acct"
		ipv6addr = ::
		port = 0
	}

	authorize {

		filter_username

		auth_log

		suffix

    ldap
		pap
		mschap

		eap {
			ok = return
			updated = return
		}
	}

	authenticate {
		Auth-Type MS-CHAP {
			mschap
		}

		eap
	}

	preacct {
		suffix
	}

	accounting {
	}

	post-auth {
		update {
			&reply: += &session-state:
		}

		reply_log
		Post-Auth-Type REJECT {
			attr_filter.access_reject

			eap

			reply_log
		}
	}

	pre-proxy {
		pre_proxy_log
		if("%{Packet-Type}" != "Accounting-Request") {
			attr_filter.pre-proxy
		}
	}

	post-proxy {
		post_proxy_log
		attr_filter.post-proxy
		eap
	}
}
```

- `/etc/freeradius/sites-enabled/vpn-inner-tunnel`:
```
server vpn-inner-tunnel {

	authorize {
		filter_username

		if ("%{request:User-Name}" =~ /^(.*)@(.*)/) {
			update request {
				Stripped-User-Name := "%{1}"
				Realm := "%{2}"
			}
		}

		auth_log
		eap

		ldap
		if ((ok || updated) && User-Password) {
			update {
				control:Auth-Type := ldap
			}
		}

		mschap
		pap
	}

	authenticate {
		Auth-Type PAP {
			pap
		}
		Auth-Type MS-CHAP {
			mschap
		}
		Auth-Type LDAP {
			ldap
		}

		eap
	}

	post-auth {
		reply_log
		Post-Auth-Type REJECT {
			attr_filter.access_reject
			reply_log
			eduroam_inner_log

			update outer.session-state {
				&Module-Failure-Message := &request:Module-Failure-Message
			}
		}
	}

	post-proxy {
		eap
	}
}
```

- `/etc/freeradius/mods-enabled/eap`:
```
eap {
	default_eap_type = peap
	timer_expire     = 60
	ignore_unknown_eap_types = no
	cisco_accounting_username_bug = no
	max_sessions = ${max_requests}
	tls-config tls-common {
		private_key_file = ${certdir}/rt.key
		certificate_file = ${certdir}/rt-chain.crt
		dh_file = ${certdir}/dh
		ca_path = ${cadir}
  cipher_list = "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-  AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA"
		cipher_server_preference = yes
		disable_tlsv1_1 = yes
		disable_tlsv1 = yes
		tls_min_version = "1.2"
		tls_max_version = "1.2"
		ecdh_curve = "prime256v1"
		cache {
			enable = yes
		}
		verify {
		}
		ocsp {
			enable = no
			override_cert_url = yes
			url = "http://127.0.0.1/ocsp/"
		}
	}
	ttls {
		tls = tls-common
		default_eap_type = pap
		copy_request_to_tunnel = yes
		use_tunneled_reply = yes
		virtual_server = "vpn-inner-tunnel"
	}
	peap {
		tls = tls-common
		default_eap_type = mschapv2
		copy_request_to_tunnel = yes
		use_tunneled_reply = yes
		virtual_server = "vpn-inner-tunnel"
	}
	mschapv2 {
	}
}
```

- `/etc/freeradius/mods-enabled/ldap`:
```
ldap {
	server = '127.0.0.1'
	port = 389
	identity = 'cn=serviceuser,ou=System,dc=example,dc=com'
	password = secret
	base_dn = 'dc=example,dc=com'
	sasl {
	}
	update {
		control:Password-With-Header	+= 'userPassword'
		control:NT-Password		:= 'sambaNTPassword'
		reply:Framed-IP-Address         := 'radiusFramedIPAddress'
		reply:Framed-IP-Netmask		:= 'radiusFramedIPNetmask'
		control:			+= 'radiusControlAttribute'
		request:			+= 'radiusRequestAttribute'
		reply:				+= 'radiusReplyAttribute'
	}
	user {
		base_dn = "ou=Users,dc=example,dc=com"
		filter = "(uid=%{%{Stripped-User-Name}:-%{User-Name}})"
		sasl {
		}
	}
	group {
		base_dn = "ou=Group,dc=example,dc=com"
		filter = '(objectClass=posixGroup)'
		membership_attribute = 'memberOf'
	}
	profile {
	}
	client {
		base_dn = "${..base_dn}"
		filter = '(objectClass=radiusClient)'
		template {
		}
		attribute {
			ipaddr				= 'radiusClientIdentifier'
			secret				= 'radiusClientSecret'
		}
	}
	accounting {
		reference = "%{tolower:type.%{Acct-Status-Type}}"
		type {
			start {
				update {
					description := "Online at %S"
				}
			}
			interim-update {
				update {
					description := "Last seen at %S"
				}
			}
			stop {
				update {
					description := "Offline at %S"
				}
			}
		}
	}
	post-auth {
		update {
			description := "Authenticated at %S"
		}
	}
	options {
		chase_referrals = yes
		rebind = yes
		res_timeout = 10
		srv_timelimit = 3
		net_timeout = 1
		idle = 60
		probes = 3
		interval = 3
		ldap_debug = 0x0028
	}
	tls {
	}
	pool {
		start = ${thread[pool].start_servers}
		min = ${thread[pool].min_spare_servers}
		max = ${thread[pool].max_servers}
		spare = ${thread[pool].max_spare_servers}
		uses = 0
		retry_delay = 30
		lifetime = 0
		idle_timeout = 60
	}
}
```

## Router configuration

#### AAA configuration
First step is the configuration of AAA.
The RADIUS server configured before is thus added to the router with the standard
commands. The IKEv2 profile will use the authentication login *aaa-vpn_auth*
and the local authorization group *local-group-author-list*.
```
aaa new-model

aaa group server radius aaa-vpn_group
 server name aaa-vpn_server

aaa authentication login aaa-vpn_auth group aaa-vpn_group
aaa authorization network local-group-author-list local

aaa attribute list attr-list1

radius server aaa-vpn_server
 address ipv4 10.0.0.2 auth-port 1812 acct-port 1813
 key secret
```

#### IKEv2 configuration
##### Authorization policy
The second step is the configuration of IKEv2. At first, a local authorization
policy *remoteaccess_auth_policy* is built with the default IP settings
that are pushed to the remote clients. Clients that don't have a static IP
address in the *radiusFramedIPAddress* LDAP attribute get an IP
from the local IPv4 pool *pool1*. This authorization policy will be sourced by
the IKEv2 profile.
```
ip local pool pool1 10.0.0.100 10.0.0.120

crypto ikev2 authorization policy remoteaccess_auth_policy
 pool pool1
 dns 8.8.8.8 8.8.4.4
 netmask 255.255.255.0
 def-domain example.com
 aaa attribute list attr-list1
 route set interface
```

##### Proposals and policy
Then, IKEv2 proposals are built with the selection of crypto algorithms. The technical guidelines for the selection are:
 - [Cisco Next Generation Cryptography](https://tools.cisco.com/security/center/resources/next_generation_cryptography).
 - [CNSA suite](https://wiki.strongswan.org/projects/strongswan/wiki/IKEv2CipherSuites#Commercial-National-Security-Algorithm-CNSA-Suite-Suite-B-Cryptographic-Suites-for-IPsec-RFC-6379).

Unfortunately, major operating systems do not 100% support those guidelines, making difficult
the creation of a policy that both respects the guidelines and supports all major operating systems.
A good list with the algorithms supported by each OS can be found in the
[MikroTik wiki](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Windows_client_configuration).

Therefore, the choice is to build two different proposals. The first proposal is the "best" option, according to NGN guidelines.
The second one supports also legacy, but still secure, algorithms, which are needed for some operating systems. T
hen, in the IKEv2 policy, the two proposals are listed in order of preference.
```
crypto ikev2 proposal ngn-proposal
 encryption aes-gcm-256
 prf sha512 sha384
 group 20 21 19
crypto ikev2 proposal remoteaccess_legacy_proposal
 encryption aes-cbc-256
 integrity sha512 sha384 sha256
 group 19 20 21 14
crypto ikev2 policy remoteaccess
 proposal ngn-proposal
 proposal remoteaccess_legacy_proposal
```

##### Profile
At the end, the IKEv2 profile *remoteaccess* is built, which combines all of the above
settings. In the profile:
 - The server is identified with a TLS certificate, included in the trustpoint *rt1.trustpoint*, which is assumed
to be already imported in the router. The client is instead identified with EAP, without AnyConnect proprietary extensions.
 - The profile is linked to the *Virtual-Template* interface 1.
 - Unfortunately, the Windows native IKEv2 client identifies itself with the local IP address of the machine, instead of the EAP identity.
  Therefore, this profile must match any possible remote identity, and the *query-identity* option is set, so that the client provides the EAP identity.
 - IKEv2 fragmentation is globally active.
 - Dead Peer Detection (DPD) is enabled.

```
crypto ikev2 fragmentation

crypto ikev2 profile remoteaccess
 match identity remote any
 identity local fqdn rt.inrim.it
 authentication local rsa-sig
 authentication remote eap query-identity
 pki trustpoint rt1.trustpoint
 dpd 60 5 on-demand
 nat keepalive 500
 aaa authentication eap aaa-vpn_auth
 aaa authorization group eap list local-group-author-list remoteaccess_auth_policy
 aaa authorization user eap cached
 virtual-template 1 mode auto
```

#### IPsec configuration
IPSec crypto configuration follows the same rules as IKEv2 configuration. Since not all OSs support PFS, it is disabled, and
algorithms are listed in order of preference, according to Cisco NGE guidelines. Anti-replay window size is
extended from 64 to 128 packets.
```
crypto ipsec transform-set AES_CBC_256_HMAC_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

 crypto ipsec profile remoteaccess
  set security-association replay window-size 128
  set transform-set AES_GCM_256 AES_CBC_256_HMAC_SHA256 AES_CBC_256_HMAC_SHA1
  set ikev2-profile remoteaccess
```

#### Interface configuration
At the end, a Virtual-Template tunnel interface is created. From this interface, Virtual-Access interfaces
are dynamically created for each user. This interface is bridged with the LAN interface of the router, and IPv4 NAT is
enabled to allow VPN users access the Internet.
```
interface Virtual-Template1 type tunnel
 ip unnumbered GigabitEthernet0/0/0
 ip nat inside
 no logging event link-status
 tunnel protection ipsec profile remoteaccess
```

## Client configuration
This VPN is tested and compatible with:
 - Windows 7 (and 8.1, 10) native client;
 - macOS native client;
 - iOS and iPadOS native client;
 - [strongSwan](https://www.strongswan.org/) for GNU/Linux and Android.

Configuration is quite straightforward for all OS. In this section, I collect the most common remarks concerning the configuration
of the VPN over those OS.

### Windows native client
By default, Windows does not negotiate strong encryption algorithms (AES256 and DH2048). To achieve this, a registry
key must be added, following [these instructions](https://wiki.strongswan.org/projects/strongswan/wiki/WindowsClients#AES-256-CBC-and-MODP2048). After the registry
key is inserted, the VPN can be added from Control Panel. By default, Windows authenticates with EAP-MSCHAPv2, and uses AES with CBC encryption, which is less efficient than GCM algorithms.
By going in the *Adapter Settings*, EAP-PEAP and EAP-TTLS (only for Windows 10) can be also configured. However, GCM algorithms cannot be chosen from the GUI.

On Windows 10, the best method to configure the VPN is the use of [Set-VpnConnectionIPsecConfiguration](https://docs.microsoft.com/en-us/powershell/module/vpnclient/set-vpnconnectionipsecconfiguration?view=win10-ps)
PowerShell cmdlet. This allows the use of more efficient GCM algorithms, and avoids the addition of the registry key. For example:
```
Add-VpnConnection -Name ExampleVPN -ServerAddress rt.example.com -AuthenticationMethod Eap -DnsSuffix example.com -EncryptionLevel Required -TunnelType Ikev2
Set-VpnConnectionIPsecConfiguration -ConnectionName "ExampleVPN" -Force -AuthenticationTransformConstants GCMAES256 -CipherTransformConstants GCMAES256 -DHGroup ECP384 -EncryptionMethod GCMAES256 -IntegrityCheckMethod SHA384 -PfsGroup None
```

### Apple macOS/iOS/iPadOS native client
Apple OSs support only EAP-MSCHAPv2, and ignore the DNS servers and suffix that are pushed to the clients. Therefore, manual DNS configuration is required at the client side.

### strongSwan
strongSwan fully supports the VPN. Strong and efficient crypto algorithms can be chosen by selecting the Suite-B-GCM-256 algorithms, without PFS:
 - IKEv2: aes256gcm16-prfsha384-ecp384
 - ESP: aes256gcm16
All distributions supports EAP-MSCHAPv2. Tunneled EAP methods are theoretically supported on GNU/Linux, but we experienced several issues with them. Therefore, we recommend
the use of MSCHAPv2 over all strongSwan (GNU/Linux and Android) installations.

Some distributions experience PMTU issues, which is a known issue of strongSwan on GNU/Linux (https://wiki.archlinux.org/index.php/StrongSwan).

#### License
Content is released under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png)