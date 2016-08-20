## This installation guide of Openvpn is used Archlinux as example, but the steps are mostly the same when applying on other linux distributions.

## Summary of Openvpn [from Archwiki](https://wiki.archlinux.org/index.php/Easy-rsa)
- A public master Certificate Authority (CA) certificate and a private key.
- A separate public certificate and private key pair for each server.
- A separate public certificate and private key pair for each client.

One can think of the key-based authentication in terms similar to that of how SSH keys work with the added layer of
a signing authority (the CA). OpenVPN relies on a bidirectional authentication strategy, so the client must
authenticate the server's certificate and in parallel, the server must authenticate the client's certificate. This
is accomplished by the 3rd party's signature (the CA) on both the client and server certificates. Once this is
established, further checks are performed before the authentication is complete.

**In the guide, all the certificates will be created on the server.**

### Do on server side, **generate a CA keypair** that will be used to sign certificates

1. install **openvpn** and **easy-rsa** packages

        # pacman -Syy --noconfirm openvpn easy-rsa

2. **initialize a new PKI** and **generate a CA keypair** that will be used to sign certificates

        # cd /etc/easy-rsa
        # easyrsa init-pki

        WARNING!!!

        You are about to remove the EASYRSA_PKI at: /etc/easy-rsa/pki
        and initialize a fresh PKI here.

        Type the word 'yes' to continue, or any other input to abort.
          Confirm removal: yes

        init-pki complete; you may now create a CA or requests.
        Your newly created PKI dir is: /etc/easy-rsa/pki


        # easyrsa build-ca

        Generating a 2048 bit RSA private key
        ...................................+..........+......................
        writing new private key to '/etc/easy-rsa/pki/private/ca.key.mvGOCfjz3O'
        Enter PEM pass phrase:<NEW PASSWORD>
        Verifying - Enter PEM pass phrase:
        -----
        You are about to be asked to enter information that will be incorporated
        into your certificate request.
        What you are about to enter is what is called a Distinguished Name or a DN.
        There are quite a few fields but you can leave some blank
        For some fields there will be a default value,
        If you enter '.', the field will be left blank.
        -----
        Common Name (eg: your user, host, or server name) [Easy-RSA CA]:foss_ca

        CA creation complete and you may now import and sign cert requests.
        Your new CA certificate file for publishing is at:
        /etc/easy-rsa/pki/ca.crt

3. Copy the CA public certificate ```/etc/easy-rsa/pki/ca.crt``` to openvpn diectory

        # cp /etc/easy-rsa/pki/ca.crt /etc/openvpn/ca.crt

### Do on server side, make the **server certificate and private key**

1. generate a **key pair** for the server, 2 files will be created ```/etc/easy-rsa/pki/reqs/<servername>.req``` & ```/etc/easy-rsa/pki/private/<servername>.key```

		# cd /etc/easy-rsa
        # easyrsa gen-req <servername> nopass    ##change the name for servername
        Generating a 2048 bit RSA private key
		......+++
		......................................................................+++
		writing new private key to '/etc/easy-rsa/pki/private/servername.key.ZTAAucWHMx'
		-----
		You are about to be asked to enter information that will be incorporated
		into your certificate request.
		What you are about to enter is what is called a Distinguished Name or a DN.
		There are quite a few fields but you can leave some blank
		For some fields there will be a default value,
		If you enter '.', the field will be left blank.
		-----
		Common Name (eg: your user, host, or server name) [<servername>]:(press enter)

		Keypair and certificate request completed. Your files are:
		req: /etc/easy-rsa/pki/reqs/<servername>.req
		key: /etc/easy-rsa/pki/private/<servername>.key

2. create the **Diffie-Hellman (DH) file**

        # openssl dhparam -out /etc/openvpn/dh.pem 2048
        Generating DH parameters, 2048 bit long safe prime, generator 2
		This is going to take a long time
		.................................................................................................+.................................................................................................................................

3. create the **HMAC key**. This will be used to add an additional HMAC signature to all SSL/TLS handshake packets. In
addition any UDP packet not having the correct HMAC signature will be immediately dropped

        # openvpn --genkey --secret /etc/openvpn/ta.key

### Do on server side, create the **client** certificate and private key

1. create 2 files ```/etc/easy-rsa/pki/reqs/<client>.req``` & ```/etc/easy-rsa/pki/private/<client>.key```

		# cd /etc/easy-rsa
		# easyrsa gen-req <clientname> nopass

		Generating a 2048 bit RSA private key
		.............................+++
		...............+++
		writing new private key to '/etc/easy-rsa/pki/private/client1.key.6VBTxEqWUR'
		-----
		You are about to be asked to enter information that will be incorporated
		into your certificate request.
		What you are about to enter is what is called a Distinguished Name or a DN.
		There are quite a few fields but you can leave some blank
		For some fields there will be a default value,
		If you enter '.', the field will be left blank.
		-----
		Common Name (eg: your user, host, or server name) [<clientname>]:(press enter)

		Keypair and certificate request completed. Your files are:
		req: /etc/easy-rsa/pki/reqs/<clientname>.req
		key: /etc/easy-rsa/pki/private/<clientname>.key

### Do on server side, **Sign the certificates** and pass them back to the server and clients

On the CA machine(Server), import and sign the certificate requests. 2 files will be generated ```/etc/easy-rsa/pki/issued/<servername>.crt``` & ```/etc/easy-rsa/pki/issued/<client>.crt```

1. import the ```.req``` files for server and cilent. **Optional step only for illustrating if there is a CA server, you have to transfer the server and client ```.req``` files for CA server to import**

        # cd /etc/easy-rsa
        # mv /etc/easy-rsa/pki/reqs/*.req /tmp
        # easyrsa import-req /tmp/<servername>.req <servername>

		The request has been successfully imported with a short name of: <servername>
		You may now use this name to perform signing operations on this request.


        # easyrsa import-req /tmp/<clientname>.req <clientname>

        The request has been successfully imported with a short name of: <clientname>
		You may now use this name to perform signing operations on this request.

2. sign the certs for server and client:

        # easyrsa sign-req server <servername>

		You are about to sign the following certificate.
		Please check over the details shown below for accuracy. Note that this request
		has not been cryptographically verified. Please be sure it came from a trusted
		source or that you have verified the request checksum with the sender.

		Request subject, to be signed as a server certificate for 3650 days:

		subject=
		    commonName                = <servername>


		Type the word 'yes' to continue, or any other input to abort.
		  Confirm request details: yes
		Using configuration from /etc/easy-rsa/openssl-1.0.cnf
		Enter pass phrase for /etc/easy-rsa/pki/private/ca.key:
		Check that the request matches the signature
		Signature ok
		The Subject's Distinguished Name is as follows
		commonName            :ASN.1 12:'<servername>'
		Certificate is to be certified until Aug 15 08:39:54 2026 GMT (3650 days)

		Write out database with 1 new entries
		Data Base Updated

		Certificate created at: /etc/easy-rsa/pki/issued/<servername>.crt


        # easyrsa sign-req client <clientname>

        You are about to sign the following certificate.
		Please check over the details shown below for accuracy. Note that this request
		has not been cryptographically verified. Please be sure it came from a trusted
		source or that you have verified the request checksum with the sender.

		Request subject, to be signed as a client certificate for 3650 days:

		subject=
			commonName                = <clientname>


		Type the word 'yes' to continue, or any other input to abort.
		  Confirm request details: yes
		Using configuration from /etc/easy-rsa/openssl-1.0.cnf
		Enter pass phrase for /etc/easy-rsa/pki/private/ca.key:<PASSWORD of CA created on first step>
		Check that the request matches the signature
		Signature ok
		The Subject's Distinguished Name is as follows
		commonName            :ASN.1 12:'client1'
		Certificate is to be certified until Aug 14 10:03:38 2026 GMT (3650 days)

		Write out database with 1 new entries
		Data Base Updated

		Certificate created at: /etc/easy-rsa/pki/issued/<clientname>.crt

3. move the certificates in place

        # cp /etc/easy-rsa/pki/issued/<servername>.crt /etc/openvpn

    Copy the client ```<client>.crt```, ```<client>.key```, ```ta.key``` to a directory first.

        # mkdir /etc/openvpn/client
        # mv /etc/easy-rsa/pki/issued/<client>.crt /etc/easy-rsa/pki/private/<client>.key /etc/openvpn/ta.key /etc/openvpn/client

### Do on server side, edit the **server configuration file**

1. Copy the example server configuration file. For other distributions, the sample config may not in the same path, please locate it by following command ```find /usr/share/ |grep 'server.conf'```

        # cp /usr/share/openvpn/examples/server.conf /etc/openvpn/server.conf

2. edit ```/etc/openvpn/server.conf```

        ca /etc/openvpn/ca.crt
        cert /etc/openvpn/server.crt
        key /etc/openvpn/server.key  # This file should be kept secret
        dh /etc/openvpn/dh.pem
        .
        tls-auth /etc/openvpn/ta.key 0
        .
        user nobody
        group nobody

### On server, edit the **client configuration file** and send it to client later

1. copy the example client configuration file. For other distributions, the sample config may not in the same path, please locate it by following command ```find /usr/share/ |grep 'client.conf'```

        # cp /usr/share/openvpn/examples/client.conf /etc/openvpn/client/client.conf

2. edit ```/etc/openvpn/client.conf```

        remote <server IP or hostname> 1194
        .
        user nobody
        group nobody
        ca /etc/openvpn/ca.crt
        cert /etc/openvpn/client.crt
        key /etc/openvpn/client.key
        .
        tls-auth /etc/openvpn/ta.key 1

### Do on server side, copy server private key to openvpn dir

1. The server private key can simply be symlinked:

		# ln -s /etc/easy-rsa/pki/private/<servername>.key /etc/openvpn/

2. Start up the openvpn server

		# openvpn /etc/openvpn/server.conf

### Do on client side, test the client connection
#### Windows

#### Android
