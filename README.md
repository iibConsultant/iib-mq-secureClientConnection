# iib-mq-secureClientConnection
IIB flow to demonstrate how to securely use an MQOutput node with userid/password and SSL.
## Scenario
NOTE: IBM key manager (strmqikm) was used to create the keystores and certificates.

1. The MQ queue manager TEST has 
    - a CMS keystore with a personal certificate with a DN=CN=TEST, OU=test, O=iib, C=CA and with a label of ibmwebspheremqtest.  This last part is no longer mandatory but it is the default.  I used a self-signed certificate so I extracted it and added it to the signer certficates portion of the IIB keystore.  The key.kdb and key.sth files were placed in /var/mqm/qmgrs/TEST/ssl (on Linux) and only made accessible to mqm (chown mqm.mqm key.* and chmod 660 key.*).  Don't make them world readable since the stash file is easily hacked.
    - a CONNAUTH specified where CHCKCLNT(REQUIRED) so all client connections need to supply a userid and password.  The CONNAUTH also specifies ADOPTCTX(YES) so that the userid that was authenticated by password is used for all further authorization checks.
    - a SVRCONN channel called IIBSSL.SVRCONN which has SSL enabled:
        - DEFINE CHANNEL(IIBSSL.SVRCONN) CHLTYPE(SVRCONN) MCAUSER(*NOACCESS) SSLCAUTH(OPTIONAL) SSLCIPH(TLS_RSA_WITH_AES_128_CBC_SHA256)
    - appropriate authorization records to allow the userid to acces the queue manager (+inq +connect) and access the queue T1 (+setall +put).

2. The following IIB configuration is created
    - a CMS keystore that allows trust of the QM certificate.  I used self-signed certificates so I added the extracted QM certificate to the signer certificates of this keystore.  This key.kdb and key.sth were placed in /var/mqsi/ssl (on Linux) and only made accessible to the integration node (chown iib.mqbrkrs key.* and chmod 660 key.*).  Don't make them world readable since the stash file is easily hacked.
    - mqsichangeproperties IIB10 -o BrokerRegistry -n mqKeyRepository -v /var/mqsi/ssl/key   (this gives the broker access to the keystore)
    - mqsisetdbparms IIB10 -n mq::idForMQ -u test -p password    (this sets up the security ID to be used for MQ.  That is, MQ authorization will be done against the userid test).

3. The following MQOutput node configuration is required on the MQ Connection tab (use appropriate values for your configuration):
    - Connection = MQ client connection properties
    - Destination queeu manager name = TEST
    - Queue manager host name = localhost   (I used a local queue manager.  Normally this would be another hostname/IP)
    - Listener port number = 1414
    - Channel name = IIBSSL.SVRCONN
    - Security identity = idForMQ
    - Use SSL = checked
    - SSL peer name = CN=TEST, OU=test, O=iib, C=CA     (this is the DN of the QM certificate.  This is probably not so important because I am using self-signed certificates but if I only had a CA certificate in my keystore, this would ensure I can only connect to the right queue manager)
    - SSL cipher specification = TLS_RSA_WITH_AES_128_CBC_SHA256   (this needs to match the value on the SVRCONN channel on the queue manager).

4. Use a Rest client to POST a message to the URL http://localhost:7080/test/SSL and invoke the flow.

5. Check the queue T1 for a message.

 

