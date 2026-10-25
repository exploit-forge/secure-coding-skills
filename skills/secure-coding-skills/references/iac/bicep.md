# Bicep

Azure misconfiguration and secret handling (A02, A03, A04). Prefer a managed identity over a password in a parameter. `@secure()` keeps a value out of deployment logs; it does not make a secret safe to commit in a `.bicepparam` file.

## Storage is not public (A02)

Set `allowBlobPublicAccess` to false. Use TLS 1.2. Disable public network access when the account should be reached only through a private endpoint. A storage account that serves a public static site is a product choice; say so and do not treat it as a data-bucket defect.

```bicep
resource data 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: accountName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
    publicNetworkAccess: 'Disabled'
  }
}
```

```bicep
properties: {
  allowBlobPublicAccess: true
  supportsHttpsTrafficOnly: false
}
```

What goes wrong: anonymous callers can read blobs, or the account accepts HTTP. Customer data does not belong on that setting.

## Network security groups name a source (A02)

Allow the app port from the virtual network or the load balancer. Do not allow every port from `*` or `Internet`.

```bicep
securityRules: [
  {
    name: 'https-from-vnet'
    properties: {
      protocol: 'Tcp'
      direction: 'Inbound'
      access: 'Allow'
      priority: 100
      sourceAddressPrefix: '10.0.0.0/16'
      sourcePortRange: '*'
      destinationAddressPrefix: '*'
      destinationPortRange: '443'
    }
  }
]
```

```bicep
{
  name: 'open'
  properties: {
    protocol: '*'
    direction: 'Inbound'
    access: 'Allow'
    priority: 100
    sourceAddressPrefix: '*'
    destinationPortRange: '*'
  }
}
```

What goes wrong: the subnet accepts any inbound connection. Management ports (22, 3389) and data ports (1433, 5432, 3306) need a named source, not `Internet`.

## Use a managed identity (A04, A07)

Give the app a system-assigned or user-assigned identity and grant that identity the role it needs. Do not pass a SQL password or a storage key in a parameter when the service supports identity.

```bicep
resource app 'Microsoft.Web/sites@2023-12-01' = {
  name: appName
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: plan.id
    httpsOnly: true
  }
}
```

```bicep
param sqlPassword string

resource app 'Microsoft.Web/sites@2023-12-01' = {
  name: appName
  properties: {
    siteConfig: {
      appSettings: [
        { name: 'SQL_PASSWORD', value: sqlPassword }
      ]
    }
  }
}
```

What goes wrong: the password lands in the template parameters, often in git, and in app settings as a readable value. The identity has no password to leak. Grant the identity a specific role on the specific resource.

## Secure parameters are not a place to store the secret (A03, A04)

If a secret must be passed, mark it `@secure()` and supply it from Key Vault at deploy time (`az.getSecret` in a `.bicepparam` file). Do not give a `@secure()` parameter a default. Do not output a secret: deployment history stores outputs.

```bicep
@secure()
param adminPassword string
```

```bicep
param adminPassword string = 'ChangeMe123'

output dbPassword string = adminPassword
```

What goes wrong: a default is in the file. An output is in the deployment log for anyone who can read the deployment. `@secure()` without a default, and no secret output, is the minimum when an identity cannot replace the password.
