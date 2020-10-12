# Day 9 - Creating an Azure Service Principal that uses Certificate Authentication (Linux Edition)

In our previous article(s) **[Day 4](https://github.com/starkfell/100DaysOfIaC/blob/master/articles/day.4.creating.a.service.principal.linux.in.plain.english.md)** and **[Day 6](https://github.com/starkfell/100DaysOfIaC/blob/master/articles/day.6.creating.a.service.principal.windows.in.plain.english.md)** we created a Service Principal with Password Authentication. Today we are going to go over how to create a Service Principal that uses a PEM Certificate for authentication using the Azure CLI on Linux.

The topic of creating Service Principal that use Certificate Authentication may seem a bit redundant; however, it is still relevant to your IaC processes as it provides you with another method of controlling deployment of resources into Azure when using tools like ARM, Ansible, or Terraform.

> **NOTE:** This article was tested and written for a Linux Host running Ubuntu 18.04.

<br />

In this installment, we'll be going over the following.

[Generate a new PEM Certificate](#generate-a-new-pem-certificate)<br />
[Create the Service Principal](#create-the-service-principal)<br />
[Test the Service Principal](#test-the-service-principal)<br />
[Updating a Service Principal Certificate](#updating-a-service-principal-certificate)<br />
[Conclusion](#conclusion)<br />

***
SPONSOR: Need to stop and start your development VMs on a schedule? The Azure Resource Scheduler let's you schedule up to 10 Azure VMs for FREE! Learn more [HERE](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/lumagatena.resourcescheduler?tab=Overview)
***

## Generate a new PEM Certificate

Make sure you are in your user's **/home** directory and run the following command to create an empty **.rnd** file for the Random Number Generation that is used by **openssl**.

```bash
touch .rnd
```

Creation of the **.rnd** file is not technically necessary since the PEM Certificate will generate irrespectively; however, we included it to prevent unnecessary confusion from seeing the error below if you don't create the **.rnd** file in your **/home** directory:

```console
Can't load /home/USERNAME/.rnd into RNG
140291203867072:error:2406F079:random number generator:RAND_load_file:
Cannot openfile:../crypto/rand/randfile.c:88:Filename=/home/USERNAME/.rnd
```

<br />

Run the command below to generate a self-signed PEM Certificate.

```bash
openssl req \
-newkey rsa:2048 \
-new \
-nodes \
-x509 \
-days 365 \
-subj "/C=AT/ST=Styria/L=Graz/O=100 Days of IaC/OU=DevOps/CN=starkfell.github.io" \
-keyout key.pem \
-out cert.pem && \
cat key.pem > iac-sp-cert.pem && \
cat cert.pem >> iac-sp-cert.pem
```

You should get back the following output below.

```console
Generating a RSA private key
..........+++++
...............................+++++
writing new private key to 'key.pem'
-----
```

If you run an **ls -lh** in your Bash Shell you should see the following files.

```console
-rw-rw-r-- 1 USERNAME USERNAME 1.4K Sep 12 09:29 cert.pem
-rw-rw-r-- 1 USERNAME USERNAME 3.1K Sep 12 09:29 iac-sp-cert.pem
-rw------- 1 USERNAME USERNAME 1.7K Sep 12 09:29 key.pem
```

Additional documentation on generating a certificate using **openssl**, can be found in its **[Official Linux Man Page](https://linux.die.net/man/1/req)**.

<br />

## Create the Service Principal

Run the following command to generate a new Service Principal using the PEM Certificate.

```bash
/usr/bin/az ad sp create-for-rbac \
--role "contributor" \
--name "iac-sp" \
--cert @iac-sp-cert.pem
```

You should get back something similar to what is shown below.

```console
Changing "iac-sp" to a valid URI of "http://iac-sp", which is the required format used for service principal
names Certificate expires 2020-09-12 09:43:25+00:00. Adjusting SP end date to match.
Creating a role assignment under the scope of "/subscriptions/00000000-0000-0000-0000-000000000000"
  Retrying role assignment creation: 1/36
  Retrying role assignment creation: 2/36
  Retrying role assignment creation: 3/36
{
  "appId": "00000000-0000-0000-0000-000000000000",
  "displayName": "iac-sp",
  "name": "http://iac-sp",
  "password": null,
  "tenant": "00000000-0000-0000-0000-000000000000"
}
```

> **NOTE:** If you attempt to add certificate authentication to an existing Service Principal with password authentication, it will only add the certificate or overwrite the existing certificate associated with the Service Principal. Any existing Password authentication will still be valid.

<br />

## Test the Service Principal

Run the command below and replace the **--tenant** value with either the **FQDN** or the **GUID** of your Azure Active Directory Tenant ID.

```bash
az login \
--service-principal \
-u http://iac-sp \
-p iac-sp-cert.pem \
--tenant 00000000-0000-0000-0000-000000000000
```

You should get back the following response verifying that you are logged in to the Azure Subscriptions as the Service Principal.

```console
[
  {
    "cloudName": "AzureCloud",
    "id": "00000000-0000-0000-0000-000000000000",
    "isDefault": true,
    "name": "{AZURE_SUBSCRIPTION_NAME}",
    "state": "Enabled",
    "tenantId": "00000000-0000-0000-0000-000000000000",
    "user": {
      "name": "http://iac-sp",
      "type": "servicePrincipal"
    }
  }
]
```

> **NOTE:** Don't forget to logout and log back in to Azure with your normal credentials after testing the Service Principal; otherwise, you will be stay logged in as the Service Principal and you may find that the rights of the Service Principal are limited compared to your own.

<br />

## Updating a Service Principal Certificate

Updating the Certificate for the Service Principal is as easy as creating a new PEM certificate and then running the command to create a new Service Principal using the commands demonstrated earlier. When the command to create a new Service Principal is ran, you'll notice the idempotence of Azure CLI as the Service Principal will be updated instead of attempting to create a new one.

```console
Changing "iac-sp" to a valid URI of "http://iac-sp", which is the required format used for service principal
names Certificate expires 2020-09-12 09:50:29+00:00. Adjusting SP end date to match.
Found an existing application instance of "00000000-0000-0000-0000-000000000000". We will patch it
Creating a role assignment under the scope of "/00000000-0000-0000-0000-000000000000"
  Role assignment already exits.

{
  "appId": "00000000-0000-0000-0000-000000000000",
  "displayName": "iac-sp",
  "name": "http://iac-sp",
  "password": null,
  "tenant": "00000000-0000-0000-0000-000000000000"
}
```

<br />

## Conclusion

If you plan on deploying IaC to the Azure Cloud using IaC Tools such as ARM, Ansible, or Terraform, you may want to consider using Certificate Based Authentication for your Service Principals as an alternative to standard Password Authentication. The steps above can be easily compiled together into a script whereby you would be able to generate new Certificates for your Service Principals on the fly for scenarios such as:

* Deploying a new Service Principals
* Resetting an expired Service Principal Certificate
* Resetting a compromised Service Principal Certificate
