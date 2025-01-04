# Achieving Configuration Parity for Dual Data Centers in Financial Institutions

## Executive Summary

Financial institutions rely on robust and secure network infrastructures to ensure uninterrupted services and meet strict regulatory standards. Operating dual data centers (DC1 and DC2) allows for seamless failover and operational rotations, ensuring continuous operations during maintenance, upgrades, or unexpected outages. Achieving configuration parity between these data centers, while accommodating their unique configurations, requires a systematic and robust approach.

This document presents a comprehensive solution based on five key pillars:

1. Golden Source: A centralized Git repository acts as the single source of truth for all configurations, eliminating ambiguity and ensuring version control of all changes.

2. Testing: Comprehensive testing validates configurations through baseline tests (ensuring fundamental requirements are met) and dynamic tests (verifying operational behavior under various conditions).

3. CI Pipeline: A continuous integration pipeline, powered by Jenkins and integrated with Itential for automation execution, ensures systematic processing and deployment of changes.

4. Transactional Deployments: Changes are applied atomically across both data centers, with built-in rollback capabilities to ensure that failed deployments do not leave systems in an inconsistent state.

5. Operating Model: Robust governance through mandatory code reviews, pull requests, and comprehensive audit trails maintains quality and compliance.

This pillar-based approach ensures consistent and reliable application of changes across both data centers while maintaining functional equivalence. It effectively prevents configuration drift, enables seamless data center rotations, and supports resilient network operations that meet the stringent requirements of financial institutions.

## Possible Technical Solution
This section provides a hypothetical solution for achieving configuration parity between two data centers. using the following tools:

- Git
- Ansible
- Jenkins
- Itential

### Overview

Configuration drift occurs when manual or untracked changes lead to inconsistencies between DC1 and DC2. Transitioning all network configurations into version-controlled code stored in a centralized Git repository mitigates this risk. Ansible serves as the orchestration tool to automatically apply and align configurations based on the golden source in Git. This ensures every change is traceable, reproducible, and consistently applied across both data centers.

### Repository Structure and Golden Source

To accommodate unique configurations for DC1 and DC2 while maintaining functional parity, the Git repository is organized to separate data center-specific variables and shared configurations:

```
network-configs/
├── ansible/
│   ├── inventories/
│   │   ├── dc1/
│   │   │   ├── group_vars/
│   │   │   │   ├── core_switches.yml
│   │   │   │   ├── edge_routers.yml
│   │   │   │   └── firewalls.yml
│   │   │   └── hosts.yml
│   │   ├── dc2/
│   │   │   ├── group_vars/
│   │   │   │   ├── core_switches.yml
│   │   │   │   ├── edge_routers.yml
│   │   │   │   └── firewalls.yml
│   │   │   └── hosts.yml
│   │   └── shared_vars/
│   ├── roles/
│   │   ├── role_paloalto/
│   │   │   ├── tasks/main.yml
│   │   │   └── templates/
│   │   └── role_cisco/
│   │       ├── tasks/main.yml
│   │       └── templates/
│   └── playbooks/
│       ├── configure_core_switches.yml
│       ├── configure_edge_routers.yml
│       └── configure_firewalls.yml
└── docs/
```

#### Key Points
The repository structure enables three benefits:

1. Data Center Flexibility: Each data center maintains its own variable sets (under dc1/ and dc2/), allowing for necessary differences in IP ranges, routing parameters, and other location-specific configurations while preserving functional equivalence.

2. Modular Adoption: The role-based structure facilitates incremental Infrastructure-as-Code adoption. Teams can begin by managing specific device types (e.g., only firewalls initially) through automation, then expand to other network components as processes mature. Common configurations are centralized under shared_vars/ to maintain consistency.

3. Change Traceability: By following GitFlow practices, all configuration changes are tied to specific branches, pull requests, and releases. This provides a clear audit trail - any configuration issue can be traced back to its originating change, release version, and associated approval history, enabling quick root cause analysis and remediation. Tools like Bitbucket's "blame" feature make this process even simpler by allowing engineers to quickly identify who made specific changes, when they were made, and which commit introduced them, providing immediate visibility into the history of any configuration line.

### Testing

The testing framework consists of two main components: baseline configuration tests that verify critical network functionality and security controls remain intact, and dynamic application-specific tests that validate end-to-end connectivity for business-critical applications.

#### Baseline Configuration Tests

Baseline tests verify critical configuration elements that must always be present across both data centers. These tests are defined in the `ansible/tests/baseline` directory and may include for example:

- Firewall baseline tests that verify:
  - Management access is properly configured
  - Core security policies are in place
  - Required services (DNS, NTP, logging) are enabled
  
- Network baseline tests that validate:
  - BGP peering and route advertisements
  - VLAN assignments and trunking

**Example baseline test playbook**

Consider the following variable file:

```yaml
---
# Management Interface Configuration
mgmt_interface:
  ip: "10.10.1.100"
  netmask: "255.255.255.0"
  state: "up"
  description: "Management Interface - DC1"
  gateway: "10.10.1.1"

# Management Access Control Configuration
mgmt_acl:
  allowed_subnets:
    - "10.10.50.0/24"  # NOC subnet
    - "10.10.51.0/24"  # Admin subnet
    - "10.10.52.0/24"  # Security Operations subnet
  ssh_enabled: true
  telnet_enabled: false
  ssh_version: "2"
  ssh_ciphers:
    - "aes256-ctr"
    - "aes192-ctr"
    - "aes128-ctr"
  idle_timeout: 900  # 15 minutes in seconds
  max_sessions: 5
  authentication:
    - "public-key"
    - "password"
```

The baseline test playbook might look like this.

```yaml
- name: Test firewall management baseline configuration
  hosts: all
  tasks:
    - name: Verify management interface configuration
      assert:
        that:
          - mgmt_interface.ip is defined
          - mgmt_interface.ip | ipv4
          - mgmt_interface.netmask == '255.255.255.0'
          - mgmt_interface.state == 'up'
        fail_msg: >
          Management interface is misconfigured:
          IP: {{ mgmt_interface.ip | default('undefined') }}
          State: {{ mgmt_interface.state | default('undefined') }}

    - name: Verify management access control list
      assert:
        that:
          - mgmt_acl is defined
          - "'10.10.50.0/24' in mgmt_acl.allowed_subnets"  # NOC subnet
          - "'10.10.51.0/24' in mgmt_acl.allowed_subnets"  # Admin subnet
          - mgmt_acl.ssh_enabled | bool
          - not mgmt_acl.telnet_enabled | bool  # Ensure telnet is disabled
        fail_msg: >
          Management ACL is misconfigured:
          Allowed subnets: {{ mgmt_acl.allowed_subnets | default([]) }}
          SSH enabled: {{ mgmt_acl.ssh_enabled | default('undefined') }}
          Telnet enabled: {{ mgmt_acl.telnet_enabled | default('undefined') }}
```
#### Application-Specific Tests

Application-specific tests validate the operational behavior of network components for specific applications. These tests can be defined in the `ansible/tests/application` directory.

For example, the following playbook validates the connectivty between a Tomcat server and a database for some critical application.

```yaml
- name: Validate Tomcat to Database connectivity for DC1
  hosts: dc1_tomcat_servers
  vars_files:
    - "../../inventories/dc1/group_vars/application_vars.yml"
  tasks:
    - name: Validate expected firewall rules exist
      assert:
        that:
          - "'ACCEPT' in (ansible_facts.iptables.rules | selectattr('chain', 'equalto', 'INPUT') | list)"
          - "'dport {{ db_port }}' in (ansible_facts.iptables.rules | selectattr('chain', 'equalto', 'INPUT') | list)"
        fail_msg: "Required firewall rules for DC1 database connectivity are missing"

- name: Validate Tomcat to Database connectivity for DC2
  hosts: dc2_tomcat_servers
  vars_files:
    - "../../inventories/dc2/group_vars/application_vars.yml"
  tasks:
    - name: Validate expected firewall rules exist
      ...
```
Given the following variable files:
```yaml
# /inventories/dc1/group_vars/application_vars.yml
tomcat_subnet: "10.1.1.0/24"
db_subnet: "10.1.2.0/24"
db_port: 5432
```

```yaml
# /inventories/dc1/group_vars/application_vars.yml
tomcat_subnet: "10.2.1.0/24"
db_subnet: "10.2.2.0/24"
db_port: 5432


```
#### Putting it all together

Tests can be explicitly registered in a manifest file, providing more control over execution:

```yaml
#/ansible/tests/registered_tests.yml
baseline_tests:
  - name: "Firewall Rules Validation"
    path: "baseline/01_firewall/rules.yml"
    required: true              # Must pass to proceed
    tags: 
      - firewall
      - security
    
  - name: "BGP Configuration Check"
    path: "baseline/02_routing/bgp.yml"
    required: true
    tags: 
      - firewall
      - security
    
application_tests:
  - name: "app-ABC Firewall Rules Validation"
    path: "app-ABC/firewall/rules.yml"
    required: true              # Must pass to proceed
    tags: 
      - app-ABC
      - firewall
```

Tests can be executed selectively using tags or names:

```bash
# Run all tests with 'firewall' tag
ansible-playbook registered_tests.yml --tags firewall

# Run specific test by name
ansible-playbook registered_tests.yml --extra-vars "test_name=Firewall Rules Validation"
```

This approach provides several benefits:
- Easy addition of new tests without modifying existing code
- Flexible organization and categorization of tests
- Control over test execution order and dependencies
- Simple inclusion/exclusion of tests based on environment or requirements


### Automation Pipeline for CI/CD

A CICD pipeline ensures that configuration updates are thoroughly tested and safely deployed to DC1 and DC2. Jenkins orchestrates the pipeline, while Itential performs the actual changes.

This is one example of how a CI pipeline could be implemented:

```groovy
pipeline {
    agent any
    
    parameters {
        string(name: 'REPO_URL', description: 'Git repository URL')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
    }

    environment {
        ANSIBLE_VAULT_PASSWORD = credentials('ansible-vault-password')
    }

    stages {
        stage('Checkout & Analysis') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.BRANCH}"]],
                        userRemoteConfigs: [[url: "${params.REPO_URL}"]]
                    ])
                    sh '''
                        # Lint YAML files and check Ansible syntax
                        find . -name "*.yml" -o -name "*.yaml" | while read file; do
                            yamllint -c .yamllint "$file" || exit 1
                        done
                        find . -name "*.yml" -path "*/playbooks/*" | while read playbook; do
                            ansible-playbook --syntax-check "$playbook" || exit 1
                        done
                    '''
                }
            }
        }

        stage('Run All Tests') {
            steps {
                script {
                    sh '''
                        # execute all tests playbooks
                    '''
                }
            }
        }
    }

    post {
        always {
            junit '**/test-results/*.xml'
            cleanWs()
            // email test results
            emailext (...)
        }
    }
}
```

### Workflow Overview

1. Engineers modify configurations in the Itential IDE and create a pull request in Bitbucket.
2. The Jenkins pipeline performs the following automated checks and tests to validate changes:
   - Linting of YAML files and syntax checks for Ansible playbooks
   - Execution of all registered tests to ensure configuration integrity
3. Approval & Merge: After all tests pass and reviews are completed, the pull request is merged into the main branch.

### Configuration Parity Solutions

Maintaining configuration parity across dual data centers involves the same core practices: storing network configurations in Bitbucket as the golden source, validating changes through Jenkins, and automating deployments with Itential and Ansible. The following two approaches illustrate distinct ways of orchestrating these components, depending on organizational preferences and tooling maturity.

#### Common Elements

Both approaches rely on the same foundational steps:
1. Bitbucket as Golden Source
Network configurations, Ansible roles, and environment variables reside in a single Git repository. Pull requests in Bitbucket capture every proposed change, with merges tracked for audit and rollback.
2.	CICD Pipelines
Teams run Jenkins pipelines when they are ready to test or deploy changes. Jenkins performs YAML linting, Ansible syntax checks, and runs baseline and application-specific tests to ensure the configurations are valid for DC1 and DC2.
3. Partity as a workflow (and Itential)
After Jenkins confirms a change is stable, Itential coordinates the final deployment to both data centers. It locks the environment to avoid collisions, manages the order of updates, and rolls back both sites if a failure occurs in either one.
4.	Rollback and Post-Deployment Checks
If a deployment fails, Bitbucket’s version history and dedicated “rollback” playbooks revert each data center to the previous known good state. Post-deployment verifications run after successful updates to confirm that DC1 and DC2 remain in sync.

Itential as the means of achiecing parity

Parity may be achieved using Itential's workflow engine. Below is an example structure (pseudocode) for a workflow that can handle dual data center deployments and rollback-on-failure. 


```yaml
workflow:
  name: "Multi-Data-Center Deployment Template"
  inputs:
    - name: dc_list
      type: array
      required: true
    - name: commit_id
      type: string
      required: true
    - name: rollback_commit_id
      type: string
      required: true

  steps:
    - name: Deploy to DC1
      action: "Run Ansible Playbook"
      inputs:
        playbook: "deploy_config_dc1.yml"
        commit_id: "{{ commit_id }}"
      onFailure:
        action: "Run Ansible Playbook"
        inputs:
          playbook: "rollback_config_dc1.yml"
          commit_id: "{{ rollback_commit_id }}"
        nextStep: Notify Failure

    - name: Deploy to DC2
      dependsOn: Deploy to DC1
      condition: Success
      action: "Run Ansible Playbook"
      inputs:
        playbook: "deploy_config_dc2.yml"
        commit_id: "{{ commit_id }}"
      onFailure:
        action: "Run Ansible Playbook"
        inputs:
          playbook: "rollback_config_both.yml"
          commit_id: "{{ rollback_commit_id }}"
        nextStep: Notify Failure

    - name: Post-Deployment Validation
      dependsOn: Deploy to DC2
      condition: Success
      action: "Run Validation Workflow"

    - name: Notify Failure
      action: "Send Notification"
      inputs:
        message: "Deployment failed. Rollback initiated."
```

#### Option A: Jenkins as the Primary Orchestrator, Itential for Multi-DC Coordination

In this model, Jenkins retains overall control of the pipeline and calls Itential only for the actual changes. After developers trigger a Jenkins job, it checks out the relevant Bitbucket branch, runs all tests, and, if everything passes, sends a deployment request to Itential. Itential then applies the new configurations to DC1 and DC2 in a coordinated, transactional manner. If any step fails—either during DC1 or DC2 updates—Itential handles the rollback, pulling the last stable commit from Bitbucket and restoring each data center to its prior configuration.

This setup keeps the CI process entirely within Jenkins while allowing Itential to focus on operational workflows like scheduling changes and orchestrating rollbacks across the data centers.

#### Option B: Ansible Automation Platform (AAP) with Itential as the Workflow Orchestrator

This approach introduces an additional layer—Ansible Automation Platform (AAP)—to run playbooks and manage inventories at scale. Jenkins still performs manual CI checks whenever a new release is ready for testing or deployment. If the changes pass validation, an engineer triggers the pipeline to update AAP job templates with the latest commit ID or version tag from Bitbucket. Itential then invokes those AAP jobs to deploy the approved changes, running them first on DC1 and then on DC2 (or in parallel, depending on defined workflows).

Integrating with AAP offers some advantages:
- **Advanced Job Scheduling**: Schedule playbook runs at specific times or intervals, ensuring updates occur during maintenance windows or low-traffic periods.
- **Role-Based Access Control**: Define and enforce permissions for different users and teams, ensuring only authorized personnel can execute or modify playbooks.
- **Inventory Management**: Maintain dynamic inventories that automatically update based on the current state of the network, reducing manual inventory tracking.
- **Centralized Logging and Reporting**: Collect and analyze logs from all playbook runs in one place, making it easier to audit changes and troubleshoot issues.
- **Scalability**: Efficiently manage and execute playbooks across a large number of devices, ensuring consistent configuration across the entire network.

### Pros and Cons of Each Approach

| Option | Pros | Cons |
|--------|------|------|
| **Option A** | - Simplicity in using Jenkins for CI processes, leveraging existing workflows with minimal changes<br>- Straightforward integration with Jenkins jobs, reducing setup complexity<br>- Itential ensures consistent deployments and rollbacks across data centers | - Limited by Jenkins' basic capabilities for job scheduling and inventory management, which may not support complex scenarios<br>- Integration with Itential can be intricate, requiring additional configuration |
| **Option B** | - Offers more capabilities with advanced job scheduling and inventory management through AAP<br>- Provides robust role-based access control and detailed job templates for enhanced security<br>- Itential's comprehensive workflow orchestration supports complex task coordination | - Increased complexity with AAP setup and maintenance, demanding more resources<br>- Steeper learning curve for teams unfamiliar with AAP, requiring training and adaptation |


## Governance

Effective governance and implementation are essential for maintaining the integrity and compliance of network configurations in financial institutions. Using GitFlow, automated scanning tools, and AI technologies, the governance framework supports development practices and ensures adherence to standards.

### GitFlow as the Operating Model

Following Gitflow and proper Git usage is now the new "operating model" for developers working on network configurations.

GitFlow provides a structured branching strategy that promotes disciplined development practices and simplifies the management of network configurations. Benefits include:

1. Branching Structure:
   - Developers work on feature branches to isolate changes from the main production branch.
   - Changes are merged into the develop branch for testing, and once validated, promoted to the main branch (eventually).
2. Code Reviews and Approvals:
   - Pull requests ensure all changes are reviewed by peers and approved before merging, which helps reduce easily preventable errors.
   - Mandatory reviews by multiple stakeholders can help drive this.
3. Git and Last Known Good Configurations:
   - During merges, GitFlow encourages squashing commits to consolidate changes into a single, meaningful commit. This practice simplifies tracking and debugging by providing a clear history of configuration changes, making it easy to identify the last known good configuration.
4. Traceability:
   - Each commit in the repository is tied to a specific feature, bug fix, or change request, ensuring every change is auditable and aligned with business objectives.

## Metrics

The following are some possible metrics that could be generated by this solution to ensure its effectiveness. These metrics can be tracked and reported by either the Itential platform or the CI/CD pipeline.

| Metric                             | Definition                                                                 | Importance                                                                                     | Implementation Suggestions                                                                 |
|------------------------------------|---------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Change Success Rate                | Percentage of configuration changes successfully deployed without rollback | Indicates effective testing and validation processes                                          | Track successful deployments versus total deployment attempts through CI/CD tools.     |
| Time to Deploy Changes             | Average time from change request initiation to successful deployment      | Identifies bottlenecks in the CI/CD pipeline                                                  | Log timestamps for change requests and deployments to calculate the average time taken.|
| Number of Configuration Drift Incidents | Instances of discrepancies between DC1 and DC2 configurations          | Assesses effectiveness in maintaining configuration parity                                     | Monitor and report drift incidents through configuration management tools. |
| Adoption Metrics                   | Rate of adoption of automated processes and tools                        | Indicates the effectiveness of training and the transition to automated workflows              | Survey teams and track usage metrics of tools like Ansible and CI/CD pipelines.         |


## Leveraging AI for Development and Governance (Available Now)

AI tools are transforming the way code is developed, reviewed, and managed. By integrating AI into governance workflows, financial institutions can improve efficiency, accuracy, and compliance.

AI for Code Generation and Assistance
1. Code Suggestions: Gen AI tools like GitHub Copilot or AWS CodeWhisperer can suggest configurations or playbook templates based on existing patterns, reducing manual effort and improving consistency.
2. Automated Unit Testing: AI can generate unit tests for Ansible playbooks, ensuring that changes are tested for correctness and stability.
3. Playbook Optimization: AI can analyze Ansible playbooks and suggest optimizations for performance and readability, ensuring they are efficient and maintainable.
4. Configuration Templates: AI can generate standardized configuration templates for common network scenarios, reducing the risk of misconfiguration and accelerating deployment.
5. Understanding the codebase: AI can understand and explain the codebase, helping new and existing developers.

AI for Code Reviews and Documentation
1. Automated Review Comments: Gen AI tools can identify potential bugs, security gaps, code smells and suggest explicit changes.
2. AI Generated documentation can be used to generate high quality documentation for the codebase.

## Why This Approach Might Work

This method keeps configurations consistent by storing them in one Git repository, so both data centers use the same setup. Versioning helps track changes, making it easy to see who did what and why, which improves accountability. Ansible's setup can handle many devices, reducing manual work and errors.

Implementing a CI/CD pipeline streamlines the deployment process, ensuring that changes are automatically tested, validated, and audited before reaching production. This reduces the risk of errors and downtime, while promoting faster delivery of updates and maintaining configuration parity across data centers.

Leveraging Gen AI tools alongside Jenkins and Itential significantly accelerates development, documentation, and testing processes. These advanced Language Models can generate, review code, and produce quality documentation, minimizing errors while raising the bar for quality.

