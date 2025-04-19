# Security Configuration Assessment

**Security Configuration Assessment (SCA)** is the process of verifying that all systems conform to a set of predefined rules regarding configuration settings and approved application usage. One of the most certain ways to secure endpoints is by reducing their vulnerability surface. This process is commonly known as **hardening**. **Configuration assessment** is an effective way to identify weaknesses in your endpoints and patch them to reduce your attack surface.

The **[Wazuh SCA]()** module performs scans to detect misconfigurations and exposures on monitored endpoints and recommend remediation actions. Those scans assess the configuration of the endpoints using policy files that contain rules to be tested against the actual configuration of the endpoint. SCA policies can check for the existence of files, directories, registry keys and values, running processes, and recursively test for the existence of files inside directories.

For example, the SCA module could assess whether it is necessary to change password-related configuration, remove unnecessary software, disable unnecessary services, or audit the TCP/IP stack configuration.

Policies for the SCA module are written in YAML. This format was chosen because it is human-readable and easy to understand. You can easily write your own SCA policies or extend existing ones to fit your needs. Furthermore, Wazuh is distributed with a set of out-of-the-box policies mostly based on the CIS benchmarks, a well-established standard for endpoint hardening.

---
CIS:  
CIS Benchmarks: These are recognized global best practices for securing IT systems and data against cyberattacks, developed by the Center for Internet Security (CIS). Wazuh supports a range of CIS Benchmarks through its Security Compliance Automation (SCA) module.  
SCA Module: This built-in Wazuh module helps assess system configurations against chosen CIS Benchmarks, generates corresponding alerts and events, and calculates a compliance score.  
  
CIS-CAT:  
Dedicated Compliance Assessment Tool: This is a separate software application, offered by CIS, specifically designed for comprehensive configuration assessment against CIS Benchmarks. It's available in both free and paid (Pro) versions.  
Integration with Wazuh: Wazuh offers integration with CIS-CAT Pro, allowing it to utilize the CIS-CAT engine for deeper scanning and reporting while leveraging Wazuh's agent deployment and alert management capabilities.  
  
Key Differences:  
Scope: CIS encompasses various benchmarks spanning different technology areas, while CIS-CAT focuses solely on configuration assessment against specific benchmarks.  
Functionality: SCA provides basic CIS compliance assessment within Wazuh, while CIS-CAT Pro offers more granular checks, vulnerability scanning (paid version), and advanced reporting features.  
Integration: While both support CIS Benchmarks, Wazuh integrates with CIS-CAT Pro to enhance its capabilities, whereas CIS-CAT stands as a separate tool.  
  
In summary, CIS refers to the broader set of best practices and benchmarks, while CIS-CAT is a specific tool for in-depth compliance assessment. Wazuh integrates with CIS-CAT Pro to combine its functionalities, providing a powerful hybrid approach for organizations prioritizing comprehensive security compliance.

---