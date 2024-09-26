# task3_scenario6


Scenario 9 - Kafka SSL Setup and Issue Resolution

Problem Statement:

The client recently performed a lift-and-shift migration of their entire platform to different machines. Following the migration, Kafka brokers and several other components were down. Upon investigating the logs, the following error was identified on the brokers:

java.lang.RuntimeException: Received a fatal error while waiting for all of the authorizer futures to be completed.
Caused by: java.util.concurrent.CompletionException: org.apache.kafka.common.errors.SslAuthenticationException: SSL handshake failed


Root Cause:

    The error indicates an SSL handshake failure, likely due to misconfigured SSL certificates or incorrect configurations in the server.properties file for the Kafka brokers. These errors need to be addressed to re-establish secure communication between the brokers and other components.

Solution Overview:

To resolve the issue, two primary tasks must be completed:

    Proper SSL Configuration for all Kafka brokers (kafka1, kafka2, kafka3) to ensure secure communication using certificates.

    Fixing Misconfigurations in the Kafka brokers' server.properties files, which include incorrect user entries and SSL principal mapping rules.

This guide walks you through the process of setting up SSL, correcting misconfigurations, and restarting the Kafka services to bring the platform back online.

Prerequisites:

    Before starting, ensure that no other versions of the Kafka sandbox are running. Use the following command to clean up any previous Docker containers and volumes:

        docker-compose down -v

    Start Kafka Sandbox:

    To begin, start the Kafka sandbox using Docker Compose:

        docker-compose up -d

    Wait for all the services to be fully up and running before proceeding to the next steps.


Step 1: Kafka SSL Setup Guide:

Objective:

To configure SSL for a Kafka cluster with three brokers (kafka1, kafka2, and kafka3). SSL (Secure Sockets Layer) ensures secure communication between Kafka brokers and clients.

    1.1 Create Your Own Certificate Authority (CA)
    
    A Certificate Authority (CA) is responsible for issuing and managing digital certificates. You’ll first create a CA to sign the SSL certificates for the Kafka brokers.

    Generate a new CA key and self-signed certificate:

        openssl req -new -x509 -days 365 -keyout ca-key -out ca-cert -subj "/C=DE/ST=NRW/L=MS/O=juplo/OU=kafka/CN=Root-CA" -passout pass:kafka-broker

        This command creates a self-signed certificate (ca-cert) valid for 365 days and a private key (ca-key).

        pass:kafka-broker is used to protect the key.

    1.2 Create Truststore and Import Root CA

    The truststore holds the public certificates of trusted entities, in this case, the Root CA.

    Create a truststore for the Kafka brokers and import the CA certificate:

        keytool -keystore kafka.server.truststore.jks -storepass kafka-broker -import -alias ca-root -file ca-cert -noprompt

        This creates a truststore (kafka.server.truststore.jks) and adds the Root CA certificate to it.

    1.3 Create Keystore for Kafka Broker

    Each Kafka broker requires its own keystore that contains the private key and the associated certificate for SSL communication.

    Generate a new keystore and private key for the Kafka broker:

        keytool -keystore kafka.server.keystore.jks -storepass kafka-broker -alias kafka1 -validity 365 -keyalg RSA -genkeypair -keypass kafka-broker -dname "CN=kafka1,OU=kafka,O=juplo,L=MS,ST=NRW,C=DE"

    1.4 Generate Certificate Signing Request (CSR)

    The CSR is a request to the CA for signing the broker’s certificate.

    Generate a CSR for the broker:

        keytool -alias kafka1 -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass kafka-broker -keypass kafka-broker

        The cert-file is the CSR that will be signed by the CA.

    1.5 Sign the Certificate Using the CA

    Sign the CSR with the CA’s certificate and key:

        openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:kafka-broker -extensions SAN -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:kafka1,DNS:localhost")

        This command signs the CSR and generates a valid certificate for kafka1, with SAN (Subject Alternative Names) including kafka1 and localhost.

    1.6 Import CA Certificate into Keystore

    Import the CA certificate into the broker’s keystore:

        keytool -importcert -keystore kafka.server.keystore.jks -alias ca-root -file ca-cert -storepass kafka-broker -keypass kafka-broker -noprompt

        This ensures the broker trusts the CA that signed its certificate.

    1.7 Import Signed Certificate into Keystore

    Finally, import the signed broker certificate into the keystore:

        keytool -keystore kafka.server.keystore.jks -alias kafka1 -import -file cert-signed -storepass kafka-broker -keypass kafka-broker -noprompt

        Now, the broker's keystore contains both the broker’s private key and the signed certificate.

    Repeat steps 1.3 to 1.7 for the other brokers (kafka2 and kafka3), replacing kafka1 with the appropriate broker names (kafka2, kafka3).


Step 2: Fix server.properties File Issues

Objective:

To correct misconfigurations in the server.properties file for all brokers.

    2.1 Incorrect User Entries

    In each broker’s server.properties file, the super.users and broker.users entries contain incorrect values. The entries use kafka-1, kafka-2, and kafka-3, but the correct identifiers are kafka1, kafka2, and kafka3.

    Incorrect lines:

        super.users=User:bob;User:kafka-1;User:kafka-2;User:kafka-3;User:mds;User:schemaregistryUser;User:controlcenterAdmin;User:connectAdmin
        broker.users=User:kafka-1;User:kafka-2;User:kafka-3
    
    Corrected lines:

        super.users=User:bob;User:kafka1;User:kafka2;User:kafka3;User:mds;User:schemaregistryUser;User:controlcenterAdmin;User:connectAdmin
        broker.users=User:kafka1;User:kafka2;User:kafka3

    2.2 Incorrect SSL Principal Mapping Rules

    Another error in the server.properties file is in the ssl.principal.mapping.rules configuration. The regular expression incorrectly includes special characters (._-) in the mapping.

    Incorrect line:

        ssl.principal.mapping.rules=RULE:^.*CN=([a-zA-Z0-9._-]*),.*$/$1/L,DEFAULT

    Corrected line:

        ssl.principal.mapping.rules=RULE:^.*CN=([a-zA-Z0-9]*),.*$/$1/L,DEFAULT

    Apply these changes to the server.properties file for all brokers (kafka1, kafka2, and kafka3).


Step 3: Restart Kafka Cluster

After applying the changes, restart the Kafka cluster using Docker Compose:

    docker-compose up -d

This will bring the services back up with the correct SSL setup and configuration changes.


Step 4: Verify the Setup

Objective:

To ensure that all services are functioning correctly and SSL issues are resolved.

    Check the logs for each broker (kafka1, kafka2, kafka3) to verify that there are no SSL handshake errors.

    Visit the Control Center at localhost:9021 to monitor the health and status of the Kafka brokers and other components.

    Ensure that all services, including Producers, Consumers, Schema Registry, Connectors, and Control Center, are working without issues.


Result

    After following the steps in this guide:

    All Kafka brokers (kafka1, kafka2, kafka3) should be running with SSL correctly configured.

    The SSL handshake errors will be resolved.

    The Kafka Control Center should show all services as operational at localhost:9021.

This setup ensures secure communication between the Kafka brokers and clients, resolving the SSL authentication issue.


This is a screenshot of the fixed issue:

![alt text](<images/Screenshot from 2024-09-25 14-59-45 (3rd copy).png>)
![alt text](<images/Screenshot from 2024-09-25 14-59-51 (3rd copy).png>)
![alt text](<images/Screenshot from 2024-09-25 14-59-51 (another copy).png>)
![alt text](<images/Screenshot from 2024-09-25 15-01-57.png>)
![alt text](<images/Screenshot from 2024-09-25 15-02-55 (copy).png>)
![alt text](<images/Screenshot from 2024-09-25 15-02-55.png>)



