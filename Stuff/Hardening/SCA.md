# Security Configuration Assessment

**Security Configuration Assessment (SCA)** is the process of verifying that all systems conform to a set of predefined rules regarding configuration settings and approved application usage. One of the most certain ways to secure endpoints is by reducing their vulnerability surface. This process is commonly known as **hardening**. **Configuration assessment** is an effective way to identify weaknesses in your endpoints and patch them to reduce your attack surface.

The **[Wazuh SCA]()** module performs scans to detect misconfigurations and exposures on monitored endpoints and recommend remediation actions. Those scans assess the configuration of the endpoints using policy files that contain rules to be tested against the actual configuration of the endpoint. SCA policies can check for the existence of files, directories, registry keys and values, running processes, and recursively test for the existence of files inside directories.

For example, the SCA module could assess whether it is necessary to change password-related configuration, remove unnecessary software, disable unnecessary services, or audit the TCP/IP stack configuration.

Policies for the SCA module are written in YAML. This format was chosen because it is human-readable and easy to understand. You can easily write your own SCA policies or extend existing ones to fit your needs. Furthermore, Wazuh is distributed with a set of out-of-the-box policies mostly based on the CIS benchmarks, a well-established standard for endpoint hardening.

