# Integration with AD #


You can configure the OP5 Log Analytics to communicate with Active Directory to authenticate users. 
To integrate with Active Directory, you configure an Active Directory realm and assign Active Directory 
users and groups to the OP5 Log Analytics roles in the role mapping file.

To protect passwords, communications between the OP5 Log Analytics and the LDAP server should be encrypted 
using SSL/TLS. Clients and nodes that connect via SSL/TLS to the LDAP server need to have the LDAP 
server’s certificate or the server’s root CA certificate installed in their keystore or truststore.
