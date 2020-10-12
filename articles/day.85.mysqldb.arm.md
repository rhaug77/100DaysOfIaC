# Day 85 - Deploying Azure DB for MySQL in Azure using ARM

One of the most widely used open source relational database platforms is MySQL. It has a storied history from it's Swedish roots, to later acquisition by Sun Microsystems, which was later acquired by Oracle. So, MySQL is open source, backed by Oracle. Microsoft some years ago hatched a new cloud-hosted model of MySQL in Azure PaaS, with their **Azure Database for MySQL** offering.

You can use the resources in this article with info available in previous installments in this series to deploy Azure Database for MySQL from a release pipeline in Azure Pipelines.

## Service Features

Azure DB for MySQL shares a service model with Azure DB for PostgreSQL and others, in that Microsoft manages the server instance, and much of the care and feeding for you. For example, OS and MySQL patching are automatic. The service includes some monitoring capability, offering the ability to set alerts and act on the database thresholds. It includes a backup feature, allowing some control of retention (ranging from 7 - 35 days) and geo-redundancy of backup storage. Your newly deployed instance even includes a firewall for the service, blocking all connections by default.

There is a JSON reference for the service at [Microsoft.DBforMySQL resource types](https://docs.microsoft.com/en-us/azure/templates/microsoft.dbformysql/allversions) which will help you track down properties, and explanation of their allowed values.

## Sample ARM Template

This is a sample template I use in the lab, and is an enhanced version of an old Azure QuickStart template, which will deploy:

- Azure DB for MySQL instance, with your choice of version 5.6, 5.7, or 8.0 (defaults to 8.0)
- An empty database (matching your web app name)
- Firewall settings on the instance with a rule to allow all Azure IPs.
- Database backups (with 7 day retention and no geo-redundancy to save costs)
- An Azure app service instance with a connection string configured to the backend database

All things considered, it's a great way to get a look at Azure Database for MySQL in a broader configuration that resembles something your developers might put together.

``` JSON
{
    "$schema": "https://schema.management.azure.com/schemas/2019-08-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "siteName": {
            "type": "string",
            "defaultValue": "contoso-web-test",
            "metadata": {
                "description": "Name of azure web app"
            }
        },
        "administratorLogin": {
            "type": "string",
            "defaultValue": "mysqladmin",
            "minLength": 1,
            "metadata": {
                "description": "Database administrator login name"
            }
        },
        "administratorLoginPassword": {
            "type": "securestring",
            "minLength": 8,
            "maxLength": 128,
            "metadata": {
                "description": "Database administrator password"
            }
        },
        "databaseSkucapacity": {
            "type": "int",
            "defaultValue": 2,
            "allowedValues": [
                2,
                4,
                8,
                16,
                32
            ],
            "metadata": {
                "description": "Azure database for PostgreSQL compute capacity in vCores (2,4,8,16,32)"
            }
        },
        "databaseSkuName": {
            "type": "string",
            "defaultValue": "GP_Gen5_2",
            "allowedValues": [
                "GP_Gen5_2",
                "GP_Gen5_4",
                "GP_Gen5_8",
                "GP_Gen5_16",
                "GP_Gen5_32",
                "MO_Gen5_2",
                "MO_Gen5_4",
                "MO_Gen5_8",
                "MO_Gen5_16",
                "MO_Gen5_32"
            ],
            "metadata": {
                "description": "Azure database for PostgreSQL sku ."
            }
        },
        "databaseSkuSizeMB": {
            "type": "int",
            "allowedValues": [
                204800,
                102400,
                51200
            ],
            "defaultValue": 51200,
            "metadata": {
                "description": "Azure database for PostgreSQL Sku Size "
            }
        },
        "databaseSkuTier": {
            "type": "string",
            "defaultValue": "GeneralPurpose",
            "allowedValues": [
                "GeneralPurpose",
                "MemoryOptimized"
            ],
            "metadata": {
                "description": "Azure database for PostgreSQL pricing tier"
            }
        },
        "mysqlVersion": {
            "type": "string",
            "allowedValues": [
                "8.0",
                "5.6",
                "5.7"
            ],
            "defaultValue": "8.0",
            "metadata": {
                "description": "MySQL version"
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]",
            "metadata": {
                "description": "Location for all resources."
            }
        },
        "databaseSkuFamily": {
            "type": "string",
            "defaultValue": "Gen5",
            "metadata": {
                "description": "Azure database for PostgreSQL sku family"
            }
        }
    },
    "variables": {
        "databaseName": "[concat(parameters('siteName'), 'database')]",
        "serverName": "[concat(parameters('siteName'), 'pgserver')]",
        "hostingPlanName": "[concat(parameters('siteName'), 'serviceplan')]"
    },
    "resources": [
        {
            "apiVersion": "2016-09-01",
            "name": "[variables('hostingPlanName')]",
            "type": "Microsoft.Web/serverfarms",
            "location": "[parameters('location')]",
            "properties": {
                "name": "[variables('hostingPlanName')]",
                "workerSizeId": "1",
                "reserved": true,
                "numberOfWorkers": 0,
                "hostingEnvironment": ""
            },
            "sku": {
                "Tier": "Standard",
                "Name": "S1"
            }
        },
        {
            "apiVersion": "2016-08-01",
            "name": "[parameters('siteName')]",
            "type": "Microsoft.Web/sites",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[concat('Microsoft.Web/serverfarms/', variables('hostingPlanName'))]"
            ],
            "properties": {
                "name": "[parameters('siteName')]",
                "serverFarmId": "[variables('hostingPlanName')]",
                "hostingEnvironment": ""
            },
            "resources": [
                {
                    "apiVersion": "2016-08-01",
                    "name": "connectionstrings",
                    "type": "config",
                    "dependsOn": [
                        "[concat('Microsoft.Web/sites/', parameters('siteName'))]"
                    ],
                    "properties": {
                        "defaultConnection": {
                            "value": "[concat('Database=', variables('databaseName'), ';Data Source=', reference(resourceId('Microsoft.DBforMySQL/servers',variables('serverName'))).fullyQualifiedDomainName, ';User Id=', parameters('administratorLogin'),'@', variables('serverName'),';Password=', parameters('administratorLoginPassword'))]",
                            "type": "MySql"
                        }
                    }
                }
            ]
        },
        {
            "apiVersion": "2017-12-01",
            "location": "[parameters('location')]",
            "name": "[variables('serverName')]",
            "type": "Microsoft.DBforMySQL/servers",
            "sku": {
                "name": "[parameters('databaseSkuName')]",
                "tier": "[parameters('databaseSkuTier')]",
                "capacity": "[parameters('databaseSkucapacity')]",
                "size": "[parameters('databaseSkuSizeMB')]",
                "family": "[parameters('databaseSkuFamily')]"
            },
            "properties": {
                "version": "[parameters('mysqlVersion')]",
                "administratorLogin": "[parameters('administratorLogin')]",
                "administratorLoginPassword": "[parameters('administratorLoginPassword')]",
                "storageProfile": {
                    "storageMB": "[parameters('databaseSkuSizeMB')]",
                    "backupRetentionDays": "7",
                    "geoRedundantBackup": "Disabled"
                },
                "sslEnforcement": "Disabled"
            },
            "resources": [
                {
                    "type": "firewallrules",
                    "apiVersion": "2017-12-01",
                    "dependsOn": [
                        "[concat('Microsoft.DBforMySQL/servers/', variables('serverName'),'/databases/' , variables('databaseName'))]",
                        "[concat('Microsoft.DBforMySQL/servers/', variables('serverName'))]"
                    ],
                    "location": "[parameters('location')]",
                    "name": "AllowAzureIPs",
                    "properties": {
                        "startIpAddress": "0.0.0.0",
                        "endIpAddress": "0.0.0.0"
                    }
                },
                {
                    "name": "[variables('databaseName')]",
                    "type": "databases",
                    "apiVersion": "2017-12-01",
                    "properties": {
                        "charset": "utf8",
                        "collation": "utf8_general_ci"
                    },
                    "dependsOn": [
                        "[concat('Microsoft.DBforMySQL/servers/', variables('serverName'))]"
                    ]
                }
            ]
        }
    ]
}
```

# MySQL Admin Tools

The go-to admin tool for MySQL is MySQL Workbench, which you can download from the MySQL website at https://www.mysql.com/products/workbench/.

***
SPONSOR: Need to stop and start your development VMs on a schedule? The Azure Resource Scheduler let's you schedule up to 10 Azure VMs for FREE! Learn more [HERE](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/lumagatena.resourcescheduler?tab=Overview)
***

## Conclusion

As you can see by now, the "Azure Database for..." family of PaaS database services makes moving your relational database workloads to the cloud easier than ever. And with comprehensive configuration available through ARM, it's easy to integrate into your Infrastructure-as-Code strategy in Azure DevOps.
