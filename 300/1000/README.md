# 1000 - Azure DevOps / CI-CD Architecture

## Overview

DevOps CI/CD architecture automates the software development lifecycle from code commit to production deployment. Continuous Integration (CI) automatically builds and tests code changes, while Continuous Deployment (CD) automatically deploys tested code to environments. Azure provides comprehensive tools for implementing DevOps practices, enabling faster delivery, higher quality, and better collaboration between development and operations teams.

## Architecture Diagram Concept

```
[Developer Workstation]
    ↓ (git push)
[Source Control: Azure Repos / GitHub]
    ↓ (trigger)
[CI Pipeline: Azure Pipelines]
    ├─→ Build Code
    ├─→ Run Tests
    ├─→ Security Scanning
    └─→ Create Artifacts
         ↓
[Artifact Repository: Azure Artifacts / Container Registry]
         ↓
[CD Pipeline: Azure Pipelines]
    ├─→ Dev Environment (Azure App Service)
    ├─→ QA Environment (Approval Gate)
    └─→ Production Environment (Blue/Green Deployment)
         ↓
[Monitoring: Azure Monitor / Application Insights]
    ↓ (feedback)
[Developer Workstation]
```

## Key Components and Their Connections

### Source Control Layer

#### 1. **Azure Repos**

- **Purpose**: Git repository hosting for source code
- **Connection**:
  - Stores application code, infrastructure as code (IaC)
  - Integrates with Azure Pipelines for CI/CD triggers
  - Branch policies enforce code review and quality gates
  - Part of Azure DevOps suite

**Azure Repos Features:**

- **Git**: Distributed version control
- **Pull Requests**: Code review with required reviewers
- **Branch Policies**: Require builds, code reviews, work item linking
- **Code Search**: Search across repositories
- **Semantic Code Search**: Understand code relationships

**Branch Strategy (GitFlow Example):**

```
main (production)
  ↑
develop (integration)
  ↑
feature/* (new features)
hotfix/* (production fixes)
release/* (release preparation)
```

#### 2. **GitHub**

- **Purpose**: Alternative Git hosting with strong community and marketplace
- **Connection**:
  - Integrates with Azure Pipelines via GitHub Actions or direct
  - GitHub Advanced Security for vulnerability scanning
  - Azure Boards integration for work item tracking
  - GitHub Copilot for AI-assisted coding

**GitHub vs Azure Repos:**

|Feature          |Azure Repos             |GitHub                   |
|-----------------|------------------------|-------------------------|
|Integration      |Native Azure DevOps     |Marketplace + Actions    |
|Pricing          |Included in Azure DevOps|Free public, paid private|
|Community        |Internal/enterprise     |Open source focus        |
|Advanced Security|Defender for DevOps     |GitHub Advanced Security |

### Build and CI Layer

#### 3. **Azure Pipelines**

- **Purpose**: CI/CD automation platform
- **Connection**:
  - Triggered by commits to Azure Repos or GitHub
  - Runs on Microsoft-hosted or self-hosted agents
  - Builds code, runs tests, creates artifacts
  - Deploys to Azure resources

**Pipeline Types:**

- **Build (CI) Pipeline**: Compiles code, runs tests, creates artifacts
- **Release (CD) Pipeline**: Deploys artifacts to environments
- **YAML Pipelines**: Pipeline as code (recommended)
- **Classic Pipelines**: Visual designer (legacy)

**Example YAML Pipeline:**

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: Build
  jobs:
  - job: BuildJob
    steps:
    - task: UseDotNet@2
      inputs:
        version: '8.x'
    
    - task: DotNetCoreCLI@2
      displayName: 'Restore dependencies'
      inputs:
        command: 'restore'
    
    - task: DotNetCoreCLI@2
      displayName: 'Build'
      inputs:
        command: 'build'
        arguments: '--configuration Release'
    
    - task: DotNetCoreCLI@2
      displayName: 'Run unit tests'
      inputs:
        command: 'test'
        arguments: '--configuration Release --collect:"Code Coverage"'
    
    - task: PublishBuildArtifacts@1
      displayName: 'Publish artifact'
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'
        artifactName: 'drop'

- stage: Deploy
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployToAzure
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure-Connection'
              appName: 'myapp-prod'
              package: '$(Pipeline.Workspace)/drop/*.zip'
```

#### 4. **Azure Pipeline Agents**

- **Purpose**: Machines that execute pipeline tasks
- **Connection**:
  - Microsoft-hosted: Managed by Azure (Windows, Linux, macOS)
  - Self-hosted: Your own machines (on-premises or cloud)
  - Agent pools organize agents
  - Scale sets for auto-scaling self-hosted agents

**Agent Types:**

- **Microsoft-Hosted**:
  - Pre-installed tools (Node, .NET, Java, Docker)
  - Fresh environment for each job
  - 1800 minutes/month free (public projects unlimited)
- **Self-Hosted**:
  - Custom tools and configurations
  - Access to internal resources
  - Persistent storage between builds
  - No usage limits

#### 5. **GitHub Actions**

- **Purpose**: CI/CD automation native to GitHub
- **Connection**:
  - YAML workflows in `.github/workflows/`
  - Triggered by GitHub events (push, PR, release)
  - Azure integration via Azure CLI action
  - Marketplace with thousands of actions

**Example GitHub Action:**

```yaml
name: Deploy to Azure

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    - name: Build
      run: dotnet build --configuration Release
    
    - name: Test
      run: dotnet test --no-build --configuration Release
    
    - name: Publish
      run: dotnet publish -c Release -o ./publish
    
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'myapp-prod'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        package: ./publish
```

### Artifact Management Layer

#### 6. **Azure Artifacts**

- **Purpose**: Package management for NuGet, npm, Maven, Python
- **Connection**:
  - Stores build outputs (libraries, packages)
  - Integrated with Azure Pipelines
  - Feeds can be public or private
  - Upstream sources proxy public packages

**Artifact Types:**

- **NuGet**: .NET packages
- **npm**: Node.js packages
- **Maven**: Java packages
- **Python**: pip packages
- **Universal Packages**: Any file type

**Benefits:**

- Version control for dependencies
- Internal package sharing
- Proxy and cache public packages
- Retention policies

#### 7. **Azure Container Registry (ACR)**

- **Purpose**: Private Docker registry for container images
- **Connection**:
  - Stores Docker images built by pipelines
  - Integrates with AKS, App Service, Container Instances
  - Geo-replication for global deployments
  - Vulnerability scanning with Defender for Containers

**ACR Features:**

- **Tasks**: Build images in Azure (no Docker daemon needed)
- **Webhooks**: Trigger actions on image push
- **Content Trust**: Sign images for verification
- **Retention Policies**: Auto-delete old images
- **Private Endpoints**: Secure access from VNet

**Example: Build and Push Container:**

```yaml
- task: Docker@2
  displayName: 'Build and push image'
  inputs:
    command: 'buildAndPush'
    containerRegistry: 'myacr'
    repository: 'myapp'
    tags: |
      $(Build.BuildId)
      latest
```

### Testing Layer

#### 8. **Azure Test Plans**

- **Purpose**: Manual and exploratory testing
- **Connection**:
  - Test case management
  - Track test execution and results
  - Integrates with Azure Boards
  - Browser-based and exploratory testing

#### 9. **Testing in Pipelines**

- **Purpose**: Automated testing as part of CI/CD
- **Testing Types**:
  - **Unit Tests**: Test individual components
  - **Integration Tests**: Test component interactions
  - **UI Tests**: Selenium, Playwright for browser testing
  - **Load Tests**: Azure Load Testing for performance
  - **Security Tests**: OWASP ZAP, SonarQube

**Example Test Task:**

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Run integration tests'
  inputs:
    command: 'test'
    projects: '**/*IntegrationTests.csproj'
    arguments: '--configuration Release'
    publishTestResults: true
```

### Security and Compliance Layer

#### 10. **Microsoft Defender for DevOps**

- **Purpose**: Security scanning for code and dependencies
- **Connection**:
  - Integrates with Azure Pipelines, GitHub Actions
  - Scans for vulnerabilities, secrets, misconfigurations
  - Results in Microsoft Defender for Cloud
  - Blocks builds with critical findings

**Security Checks:**

- **Static Code Analysis**: SonarQube, ESLint, StyleCop
- **Dependency Scanning**: Detect vulnerable packages
- **Secret Scanning**: Find exposed API keys, passwords
- **Container Scanning**: Scan images for CVEs
- **IaC Scanning**: Terraform, ARM template validation

#### 11. **Azure Key Vault**

- **Purpose**: Secure storage for secrets, keys, certificates
- **Connection**:
  - Pipelines reference secrets from Key Vault
  - Variable groups link to Key Vault
  - No secrets in code or pipeline YAML
  - Managed identity for authentication

**Example: Use Key Vault Secret:**

```yaml
variables:
- group: 'production-secrets' # Variable group linked to Key Vault

steps:
- task: AzureCLI@2
  inputs:
    azureSubscription: 'Azure-Connection'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      echo "Using secret: $(database-password)" # Secret from Key Vault
```

#### 12. **Azure Policy**

- **Purpose**: Enforce compliance and governance
- **Connection**:
  - Policies evaluated during deployment
  - Can block non-compliant deployments
  - Audit and remediate existing resources
  - Custom policies with ARM templates

**Example Policies:**

- Require tags on resources (CostCenter, Environment)
- Enforce naming conventions
- Require encryption for storage accounts
- Block public IP addresses on VMs
- Require SSL/TLS for web apps

### Deployment Layer

#### 13. **Azure App Service**

- **Purpose**: PaaS hosting for web applications
- **Connection**:
  - Deployment slots for staging/production
  - Continuous deployment from Git or pipelines
  - Application settings from Key Vault
  - Auto-scaling based on load

**Deployment Strategies:**

- **Blue/Green**: Deploy to staging slot, swap to production (zero downtime)
- **Canary**: Gradually shift traffic to new version
- **Rolling**: Update instances one at a time
- **A/B Testing**: Split traffic between versions

**Example Deployment:**

```yaml
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'Azure-Connection'
    appName: 'myapp-prod'
    deployToSlotOrASE: true
    slotName: 'staging'
    package: '$(Pipeline.Workspace)/drop/*.zip'

- task: AzureAppServiceManage@0
  displayName: 'Swap slots'
  inputs:
    azureSubscription: 'Azure-Connection'
    action: 'Swap Slots'
    webAppName: 'myapp-prod'
    sourceSlot: 'staging'
    targetSlot: 'production'
```

#### 14. **Azure Kubernetes Service (AKS)**

- **Purpose**: Container orchestration for microservices
- **Connection**:
  - Pipelines build images, push to ACR
  - Deploy to AKS using kubectl or Helm
  - GitOps with Flux or Argo CD
  - Canary deployments with service meshes

**AKS Deployment Strategies:**

- **Rolling Update**: Default K8s strategy
- **Blue/Green**: Deploy new version alongside old, switch traffic
- **Canary**: Gradually increase traffic to new version
- **A/B Testing**: Route users based on attributes

**Example AKS Deployment:**

```yaml
- task: KubernetesManifest@0
  displayName: 'Deploy to AKS'
  inputs:
    action: 'deploy'
    kubernetesServiceConnection: 'aks-connection'
    namespace: 'production'
    manifests: |
      k8s/deployment.yaml
      k8s/service.yaml
    containers: 'myacr.azurecr.io/myapp:$(Build.BuildId)'
```

#### 15. **Azure Functions**

- **Purpose**: Serverless compute for event-driven code
- **Connection**:
  - Deploy via zip deployment or container
  - Deployment slots for staging
  - Configuration from App Settings or Key Vault
  - Continuous deployment from pipelines

#### 16. **Azure Virtual Machines**

- **Purpose**: IaaS compute for legacy applications
- **Connection**:
  - Custom VM images with Packer
  - Configuration management with Ansible, Chef, Puppet
  - Deployment via ARM templates or Terraform
  - Update management with Azure Automation

### Infrastructure as Code (IaC)

#### 17. **Azure Resource Manager (ARM) Templates**

- **Purpose**: Declarative infrastructure definition
- **Connection**:
  - JSON templates define Azure resources
  - Deployed via Azure Pipelines
  - Idempotent deployments
  - Native Azure support

**Example ARM Deployment:**

```yaml
- task: AzureResourceManagerTemplateDeployment@3
  inputs:
    deploymentScope: 'Resource Group'
    azureResourceManagerConnection: 'Azure-Connection'
    subscriptionId: '$(subscriptionId)'
    action: 'Create Or Update Resource Group'
    resourceGroupName: 'myapp-rg'
    location: 'East US'
    templateLocation: 'Linked artifact'
    csmFile: 'infrastructure/azuredeploy.json'
    csmParametersFile: 'infrastructure/azuredeploy.parameters.json'
    deploymentMode: 'Incremental'
```

#### 18. **Bicep**

- **Purpose**: Domain-specific language for ARM templates
- **Connection**:
  - Simpler syntax than ARM JSON
  - Transpiles to ARM templates
  - Strong typing and IntelliSense
  - Modular with reusable modules

**Example Bicep:**

```bicep
param location string = resourceGroup().location
param appName string

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${appName}-plan'
  location: location
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
}

resource webApp 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
  }
}
```

#### 19. **Terraform**

- **Purpose**: Multi-cloud IaC tool
- **Connection**:
  - HCL (HashiCorp Configuration Language)
  - State stored in Azure Storage
  - Supports Azure, AWS, GCP
  - Terraform tasks in Azure Pipelines

**Example Terraform Pipeline:**

```yaml
- task: TerraformInstaller@0
  inputs:
    terraformVersion: 'latest'

- task: TerraformTaskV4@4
  displayName: 'Terraform init'
  inputs:
    command: 'init'
    backendServiceArm: 'Azure-Connection'
    backendAzureRmResourceGroupName: 'terraform-state-rg'
    backendAzureRmStorageAccountName: 'tfstate'
    backendAzureRmContainerName: 'tfstate'

- task: TerraformTaskV4@4
  displayName: 'Terraform plan'
  inputs:
    command: 'plan'
    environmentServiceNameAzureRM: 'Azure-Connection'

- task: TerraformTaskV4@4
  displayName: 'Terraform apply'
  inputs:
    command: 'apply'
    environmentServiceNameAzureRM: 'Azure-Connection'
```

### Monitoring and Feedback Layer

#### 20. **Azure Monitor**

- **Purpose**: Comprehensive monitoring solution
- **Connection**:
  - Collects metrics and logs from all resources
  - Alerts on anomalies and thresholds
  - Dashboards for visualization
  - Integration with pipelines for health checks

#### 21. **Application Insights**

- **Purpose**: Application performance monitoring (APM)
- **Connection**:
  - Auto-instrumentation for App Service, Functions, AKS
  - Distributed tracing across services
  - Live metrics stream during deployments
  - Availability tests and synthetic monitoring

**Deployment Health Monitoring:**

```yaml
- task: AzureMonitor@1
  displayName: 'Query Application Insights'
  inputs:
    azureSubscription: 'Azure-Connection'
    queryType: 'application-insights'
    applicationInsightsResourceId: '/subscriptions/.../components/myapp-insights'
    query: |
      requests
      | where timestamp > ago(5m)
      | summarize errorRate = 100.0 * countif(success == false) / count()
    timeRange: '5m'
    failOnWarning: true
```

#### 22. **Azure DevOps Analytics**

- **Purpose**: Metrics and insights on DevOps practices
- **Connection**:
  - Pipeline success/failure rates
  - Lead time and deployment frequency
  - Change failure rate
  - DORA metrics

### Project Management Layer

#### 23. **Azure Boards**

- **Purpose**: Agile project management and work item tracking
- **Connection**:
  - Backlog management (epics, features, user stories, tasks)
  - Sprint planning and tracking
  - Kanban boards
  - Integration with repos (link work items to commits)

**Work Item Types:**

- **Epic**: Large body of work (quarters)
- **Feature**: Deliverable functionality (sprints)
- **User Story**: User-facing functionality
- **Task**: Implementation work
- **Bug**: Defects and issues

#### 24. **Azure DevOps Wiki**

- **Purpose**: Documentation and knowledge sharing
- **Connection**:
  - Markdown-based wiki
  - Code-backed or project wiki
  - Version controlled
  - Link to work items and repos

## DevOps Architecture Patterns

### Pattern 1: Trunk-Based Development

```
Developers → Frequent commits to main → CI runs → CD to prod
```

**Characteristics:**

- Short-lived feature branches (hours to 1 day)
- Feature flags for incomplete features
- Requires strong CI and testing
- Fast feedback loop

### Pattern 2: GitFlow

```
feature/* → develop → release/* → main (production)
```

**Characteristics:**

- Long-lived develop and main branches
- Feature branches for new work
- Release branches for stabilization
- Hotfix branches for urgent fixes

### Pattern 3: Multi-Stage Pipelines

```
CI → Dev → QA (manual approval) → Staging → Prod (manual approval)
```

**Characteristics:**

- Progressive deployment across environments
- Approval gates at critical stages
- Smoke tests between stages
- Rollback capability

### Pattern 4: Infrastructure Pipeline

```
IaC Code → Validate → Plan → Apply (manual approval) → Monitor
```

**Characteristics:**

- Separate pipeline for infrastructure changes
- Terraform/Bicep state management
- Drift detection and remediation
- Approval before production changes

### Pattern 5: Microservices CI/CD

```
Service A Code → Build A → Deploy A
Service B Code → Build B → Deploy B (independent)
```

**Characteristics:**

- Independent pipelines per service
- Decoupled deployment schedules
- Service-level canary deployments
- Shared pipeline templates

## Real-World CI/CD Flow Example

### Web Application Deployment Flow:

1. **Developer** commits code to feature branch in Azure Repos
1. **Pull Request** created targeting develop branch
1. **Branch Policy** triggers:
- Build validation pipeline runs
- Unit tests executed
- Code coverage check (>80% required)
- Static code analysis (SonarQube)
- Reviewers assigned automatically
1. **Code Review** completed, PR approved
1. **PR Merged** to develop branch
1. **CI Pipeline** triggered on develop:
- Restore dependencies
- Build application
- Run unit and integration tests
- Security scanning (Defender for DevOps)
- Build Docker image
- Push image to ACR with tag `develop-$(Build.BuildId)`
- Publish artifacts
1. **CD Pipeline** triggered automatically:
- Deploy to Dev environment (App Service slot)
- Run smoke tests
- If passed, move to QA
1. **QA Environment**:
- Manual approval required
- Deploy to QA App Service slot
- Run automated UI tests (Selenium)
- QA team manual testing
1. **Staging Environment**:
- Manual approval from product owner
- Deploy to staging slot
- Run load tests (Azure Load Testing)
- Security scanning (OWASP ZAP)
1. **Production Deployment**:
- Manual approval from DevOps lead
- Deploy to production staging slot
- Validate Application Insights metrics
- Slot swap (blue/green deployment)
- Monitor for 15 minutes
- If error rate > 5%, auto-rollback
1. **Post-Deployment**:
- Work items automatically moved to Done
- Release notes generated
- Stakeholders notified
- Metrics updated in Azure DevOps Analytics

## Best Practices

### CI Best Practices:

1. **Fast Builds**: Keep builds under 10 minutes
1. **Build Once**: Build artifact once, deploy many times
1. **Fail Fast**: Run fast tests first, slow tests later
1. **Parallel Jobs**: Run independent jobs in parallel
1. **Caching**: Cache dependencies between builds
1. **Clean Builds**: Start with clean environment
1. **Build Notifications**: Alert team on build failures

### CD Best Practices:

1. **Deployment Slots**: Use staging slots before production
1. **Gradual Rollout**: Canary or blue/green deployments
1. **Automated Rollback**: Rollback on health check failures
1. **Approval Gates**: Manual approvals for production
1. **Environment Parity**: Keep environments as similar as possible
1. **Configuration Management**: Externalize configuration
1. **Deployment Monitoring**: Watch metrics during deployment

### Security Best Practices:

1. **Secret Management**: Use Key Vault, never commit secrets
1. **Least Privilege**: Service principals with minimal permissions
1. **Dependency Scanning**: Scan for vulnerable packages
1. **Container Scanning**: Scan images before deployment
1. **Policy Enforcement**: Azure Policy for compliance
1. **Audit Logs**: Track all deployments and changes
1. **Network Security**: Deploy through private endpoints

### Testing Best Practices:

1. **Test Pyramid**: Many unit tests, fewer integration/UI tests
1. **Test Data**: Use synthetic test data
1. **Continuous Testing**: Tests run on every commit
1. **Test Isolation**: Tests don’t depend on each other
1. **Performance Tests**: Regular load testing
1. **Shift Left**: Test early in development cycle

## Common Challenges

1. **Slow Pipelines**:
- Solution: Parallel jobs, caching, incremental builds
1. **Flaky Tests**:
- Solution: Retry logic, better test isolation, async handling
1. **Environment Drift**:
- Solution: IaC, immutable infrastructure, containers
1. **Merge Conflicts**:
- Solution: Small PRs, frequent integration, feature flags
1. **Secret Management**:
- Solution: Key Vault, variable groups, no secrets in code
1. **Cost Control**:
- Solution: Self-hosted agents, optimize pipeline minutes, clean up resources

## Cost Optimization

1. **Pipeline Minutes**:
- Free tier: 1,800 minutes/month
- Self-hosted agents: No usage limits
- Parallel jobs: More expensive, use judiciously
1. **Self-Hosted Agents**:
- Run on existing VMs or Kubernetes
- No per-minute charges
- Better for high-volume builds
1. **Artifact Storage**:
- Retention policies to delete old artifacts
- Compress artifacts before publishing
1. **Testing**:
- Unit tests in CI, expensive tests in nightly builds
- Cloud-based testing (Selenium Grid) can be costly

## When to Use CI/CD

✅ **Essential for:**

- Modern application development
- Microservices architectures
- Cloud-native applications
- Teams practicing Agile/DevOps
- Applications with frequent releases

❌ **May not need for:**

- One-time scripts or tools
- Legacy applications with rare updates
- Very small teams (1-2 developers)
- Applications with manual deployment requirements

## Learning Resources

- Microsoft Learn: [Azure DevOps](https://learn.microsoft.com/azure/devops/)
- Azure Architecture Center: [CI/CD for Azure](https://learn.microsoft.com/azure/architecture/example-scenario/apps/devops-dotnet-webapp)
- [Azure Pipelines documentation](https://learn.microsoft.com/azure/devops/pipelines/)
- [GitHub Actions documentation](https://docs.github.com/actions)
- [Azure DevOps Labs](https://azuredevopslabs.com/)
- [Infrastructure as Code (Terraform/Bicep)](https://learn.microsoft.com/azure/architecture/guide/devops/infrastructure-as-code)
