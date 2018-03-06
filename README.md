Configuring Apache for OCSP Stapling...

1) Deployed vanilla RHEL7 VM


2) Installed httpd and mod_ssl
  # yum install httpd
  # yum install mod_ssl


3) Add firewall rules to allow http and https incoming traffic
  # firewall-cmd --permanent --add-service=http
  # firewall-cmd --permanent --add-service=https
  # firewall-cmd --reload
  # systemctl start httpd.service
  # systemctl status httpd.service


4) Set hostname and update /etc/hosts
  # hostnamectl set-hostname delorean.pki.com
  [add host and ip to /etc/hosts]


5) Create a private key in preparation for a certificate request.
  # mkdir -p /etc/ssl/private
  # openssl req -newkey rsa:2048 -nodes -keyout /etc/ssl/private/privateKey.key


6) Create openssl cert request configuration for a representative SSL certificate.
  # vi /etc/pki/tls/cert.cfg
    [ req ]
    default_bits = 2048
    default_keyfile =
    distinguished_name = req_distinguished_name
    [ req_distinguished_name ]
    C = Country Name (Press Enter)
    C_default = US
    C_min = 2
    C_max = 2
    O = Press Enter
    O_default = U.S. Government
    0.OU=OU=NSS Press Enter
    0.OU_default = NSS
    1.OU =Agency
    1.OU_default =
    2.OU = SubAgency
    2.OU_default =
    3.OU = SubSubAgency
    3.OU_default =
    commonName = Public FQDN of server
    commonName_max = 64


7) Create a CSR in preparation for signing
  # openssl req -out /etc/pki/tls/certs/certReq.csr -key /etc/pki/tls/private/privateKey.key -new -config /etc/pki/tls/cert.cfg


8) Submit request to CA


9) Retrieve cert from CA and copy it to /etc/pki/tls/certs/signedCert.cer


10) Retrieve chain certs and add to /etc/pki/tls/certs/server-chain.cer


11) Update the ssl.conf file (/etc/httpd/conf.d/ssl.conf)
    SSLCertificateFile [PATH TO CERTIFICATE]
    SSLCertificateKeyFile [PATH TO KEY]
    SSLCertificateChainFile /etc/pki/tls/certs/server-chain.cer
    ServerName [SERVER_NAME]:443


12) Verify the configuration
  # httpd -t
  # apachectl configtest
  # apachectl restart
  # httpd -D DUMP_VHOSTS
  # openssl s_client -connect delorean.pki.com:443 -showcerts

  Note: You'll be able to connect - but it will complain about self-signed certs in the chain due to the root.


13) Test OCSP
  # openssl ocsp -issuer /etc/pki/tls/certs/server-chain.cer -cert /etc/pki/tls/certs/signedCert.cer -url http://ocsp.nsn0.rcvs.nit.disa.mil -CAfile /etc/pki/tls/certs/server-chain.cer
  Response verify OK
  /etc/pki/tls/certs/signedCert.cer: good
  	This Update: Jul 12 09:00:00 2017 GMT
  	Next Update: Jul 18 09:00:00 2017 GMT


14) Configure OCSP Stapling
  Add the following to the  ssl.conf file (/etc/httpd/conf.d/ssl.conf)

  Outside of the virtual host configs
    SSLStaplingCache shmcb:/tmp/stapling_cache(128000)

  Inside the virtual host context configs (<VirtualHost _default_:443>)
    SSLUseStapling on


15) Test OCSP Stapling
  # apachectl restart

  # echo QUIT | openssl s_client -connect delorean.pki.com:443 -status 2> /dev/null | grep -A 17 'OCSP response:' | grep -B 17 'Next Update'
   OCSP response:
   ======================================
   OCSP Response Data:
       OCSP Response Status: successful (0x0)
       Response Type: Basic OCSP Response
       Version: 1 (0x0)
       Responder Id: 20C36780756ADFDCDDBA35643B61A2E26B3A79A7
       Produced At: Jul 12 08:00:05 2017 GMT
       Responses:
       Certificate ID:
         Hash Algorithm: sha1
         Issuer Name Hash: E9B074B1A249F72BAD75BADFD348C5CBF439199C
         Issuer Key Hash: E4505CBFC0A95D18F7F9436964563607A5997ACC
         Serial Number: 044D
       Cert Status: good
       This Update: Jul 12 01:00:00 2017 GMT
       Next Update: Jul 18 01:00:00 2017 GMT


16) Test Again
  # echo QUIT | openssl s_client -connect delorean.pki.com:443 -status | grep "OCSP Response Status:"
  depth=3 C = US, O = U.S. Government, OU = NSS, OU = Certification Authorities, CN = NSS JITC Root CA 2
  verify error:num=19:self signed certificate in certificate chain
  verify return:0
      OCSP Response Status: successful (0x0)
  DONE

  If compared to the output before stapling - you now see the OCSP response included in the sclient connection.

  # echo QUIT |openssl s_client -connect delorean.pki.com:443 -status | grep "OCSP Response Status:"
  depth=3 C = US, O = U.S. Government, OU = NSS, OU = Certification Authorities, CN = NSS JITC Root CA 2
  verify error:num=19:self signed certificate in certificate chain
  verify return:0
  DONE
