This covers how to create AKS cluster, bind it to Azure Application Gateway and use multiple ingresses in cluster with it.

On one hand, it's more or less covered in Microsoft's own manuals, on the other - it seems that there is no direct claim that you can use default managed settings to achieve multi-tenancy. Also, my implementation does this with terraform (and also some helm for ingress configuration inside bigger app chart).

In the end we will have following structure, assuming we have 2 tenants in cluster:
- 2 domains (1 for each tenant)
- 1 Application Gateway (AGW)
- 1 AKS cluster
- 1 ingress controller in said cluster
- 2 namespaces, 1 for each tenant, containing 1 instance of deployed app each (including configured ingress).

We will set up secure https connection (and redirect all http to it). For this Azure Key Vault will be used to store the certificate.

I'd like to note that we are using separate ingresses as we deploy ingress in scope of application with helm. And managing one ingress to target multiple applications managed in different helm releases seems like overcomplication or a lot of manual work.

For this example we're considering that all settings for two tenants are identical, except the domain name. Also, domain names are managed externally because that's what I've been working with. 

# AGIC
First we need to know about AGIC. MS documentation gives us the overview: [What is Azure Application Gateway Ingress Controller? | Microsoft Learn](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview):
> The Application Gateway Ingress Controller (AGIC) is a Kubernetes application, which makes it possible for [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/services/kubernetes-service/) customers to leverage Azure's native [Application Gateway](https://azure.microsoft.com/services/application-gateway/) L7 load-balancer to expose cloud software to the Internet.

To put it plainly, it's a "magic" (*agic-magic, get it?*) thing that translates ingress spec into AGW entities, such as listeners, routing rules etc.

It can be installed in 2 ways: 
- "managed", or AKS add-on - you just put a checkmark in AKS cluster network settings and select the AGW to connect to, Azure then installs everything automatically in `kube-system` namespace. This is the method I'm going to use.
- "manual" - using helm and [Microsoft manual](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-install-new). I've tried this method, maybe I didn't go too deep enough, but at the moment I'm writing this, the manual seems a bit outdated and doesn't cover some issues I've faced.

Why did I try "manual"/helm approach in the first place?  It didn't seem clear from MS manuals that AKS add-on can support multiple ingresses. To be fair, there is [Enable multiple namespace support for Application Gateway Ingress Controller | Microsoft Learn](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-multiple-namespace-support) document, but it's refers specific AGIC version, mentions helm and I got the impression that this uses 1 ingress for multiple namespaces. ~~Maybe I'm just not attentive enough. or maybe Microsoft's documentation is not clear.~~ Well, I've found [this](https://learn.microsoft.com/en-us/samples/azure-samples/aks-multi-tenant-agic/aks-multi-tenant-agic/) article after writing that states what AGIC can and cannot rather clearly, but it doesn't describe terraform, so mine is still better!

So, AKS addon it is.

Note: AKS addon supports only "1 AGW to 1 AKS cluster" relation. More complex setups need different solutions.

# Prerequisites

We need:
- terraform installed and configured with `azurerm` provider that has access to the subscription etc. where we are deploying our cluster
- public IP (for sake of example 11.22.33.44)
- domain names that have `A` records configured that point at above IP, like
	- `customer1.example.com -> A -> 11.22.33.44`
	- `customer2.example.com -> A -> 11.22.33.44`
- Wildcard certificate for `*.example.com` in a Key Vault storage.

I'll assume that all resources, both existing and to-be-created, are located in same region and skip most possible security/networking issues you can face.

# Terraform
Let's deploy stuff!

I assume that variables such as `var.customer` and others are provided externally and don't need clarification. Same goes for some terraform objects like resource groups, that are definitively required, but don't carry significance to the topic.
## Application Gateway (AGW)

First, we need to deploy AGW. In short, configuration looks like this:

```terraform
resource "azurerm_application_gateway" "agw" {
  name                = "agw"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "appGatewayIpConfig"
    subnet_id = azurerm_subnet.agw_subnet.id
  }

  frontend_port {
    name = "agw-fe-ip-port"
    port = 80
  }

  frontend_ip_configuration {
    name                 = "agw-fe-ip-config"
    public_ip_address_id = azurerm_public_ip.ip.id
  }

  request_routing_rule {
    name                       = "routing-rule"
    priority                   = 1
    rule_type                  = "Basic"
    http_listener_name         = "agw-https-listener"
    backend_address_pool_name  = "agw-be-pool"
    backend_http_settings_name = "agw-be-http-config"
  }

  http_listener {
    name                           = "agw-https-listener"
    frontend_ip_configuration_name = "agw-fe-ip-config"
    frontend_port_name             = "agw-fe-ip-port"
    protocol                       = "Http"
  }

  backend_http_settings {
    name                  = "agw-be-http-config"
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 1
  }

  backend_address_pool {
    name = "agw-be-pool"
  }

  identity {
    identity_ids = [azurerm_user_assigned_identity.uami_key_vault.id]
    type = "UserAssigned"
  }

  ssl_certificate {
    name                = "wildcard_certificate"
    key_vault_secret_id = azurerm_key_vault_certificate.wildcard.key_vault_secret_id
  }

  lifecycle {
    ignore_changes = [
      tags,
      backend_address_pool,
      backend_http_settings,
      http_listener,
      probe,
      request_routing_rule,
      frontend_port,
      redirect_configuration
    ]
  }
}
```

Let's check out what happens here.

### Routing settings

You can't create `azurerm_application_gateway` without a set of default entities related to routing, such as:  `request_routing_rule`, `http_listener`, `backend_http_settings`, `backend_address_pool`. Thing is, the ones listed here will be discarded by AGIC, so we're doing this just to satisfy Terraform  on validation stage.

Of course, there also are obligatory `frontend_ip_configuration` and `frontend_port` which we can safely set; we should attach an existing public IP to AGW:
```terraform
resource "azurerm_application_gateway" "agw" {
...
  frontend_ip_configuration {
    name                 = "agw-fe-ip-config"
    public_ip_address_id = azurerm_public_ip.agw_ip.id
  }
```

### Lifecycle
When AGIC starts to manage this AGW, it will remove our created entities and create even more of its own. This means that some time in the future when we run `terraform plan` we will get a lot of unwanted changes. AGIC can fix that automatically, but it is still a service disruption. And this is how we avoid this:
```terraform
resource "azurerm_application_gateway" "agw" {
...
  lifecycle {
    ignore_changes = [ tags,
      backend_address_pool, backend_http_settings,
      http_listener, probe, request_routing_rule,
      frontend_port, redirect_configuration ]
  }
```

It does what it seems to do: makes terraform ignore changes in mentioned blocks. Yes, `tags` are affected too, because AGIC puts there something like `managed-by-k8s-ingress : 1.7.2/12345abcde/YYYY-MM-DD-hh:mmT+0000`. Probably not worth ignoring it if we are going to use tags.

### Certificate
For https access we need a certificate. They can be imported directly to AGW, but we'll use the one that we already have in Key Vault

```terraform
resource "azurerm_application_gateway" "agw" {
...
  ssl_certificate {
    name                = "wildcard_certificate"
    key_vault_secret_id = azurerm_key_vault_certificate.wildcard_cert.key_vault_secret_id
  }
...
```

Certificate name can be more or less any string, and is used for reference inside AGW, while this same certificate can really have another name in Key Vault where it's stored.

Here, we reference terraform object `azurerm_key_vault_certificate.wildcard_cert`, but we can just paste a string variable containing secret ID, like `https://<keyvault>.vault.azure.net/secrets/<key_vault_secret_id>`. I had to do like this, because we were referencing a Key Vault outside of terraform state's scope ^[obviously the correct way is to use data source in terraform]. Note that while it's a certificate, we're referencing specifically a `secret`.


However, ~~one cannot simply~~ AGW does not have permissions to get the certificate. To provide this capability, we assign a managed identity to AGW:
```terraform
resource "azurerm_application_gateway" "agw" {
...
  identity {
    identity_ids = [azurerm_user_assigned_identity.uami_key_vault.id]
    type = "UserAssigned"
  }
...
```

And the User-Assigned Managed Identity (UAMI) object is created like this, separately from `azurerm_application_gateway`:
```terraform
resource "azurerm_user_assigned_identity" "uami_key_vault" {
  location            = azurerm_resource_group.main.location
  name                = "uami_key_vault"
  resource_group_name = azurerm_resource_group.main.name
}
```

With access policies set in Key Vault:
```terraform
resource "azurerm_key_vault" "main" {
...
  access_policy { # UAMI for AKS/AGW to Key Vault interaction
    tenant_id = var.tenant
    object_id = azurerm_user_assigned_identity.uami_key_vault.principal_id
    certificate_permissions = [
      "Get",
      "GetIssuers",
      "List",
      "ListIssuers"]
```
These should be enough.

As a result, we have AGW, that is assigned an identity object, which has permissions to look inside specific Key Vault and get a certificate from it.

## AKS cluster

Much simpler configuration here. Skipping what's not related to AGW:
```terraform
resource "azurerm_kubernetes_cluster" "aks_cluster" {
...
ingress_application_gateway {
    gateway_id = azurerm_application_gateway.k8singress_agw.id
  }
```



# Kubernetes


Assuming we have our cluster deployed and running, without any business apps deployed yet  we can already check what we have in `kube-system` namespace:
```
$ kubectl get deployment --all-namespaces -l app=ingress-appgw
kube-system            ingress-appgw-deployment              1/1     1            1           6d5h
```
This is a deployment controlling respective replicaset+pod. There is also an `ingressClass`, and a bunch of `workloadIdentity`-related objects. I'm not a specialist here, so I'll direct you to MS documentation.

Remember, `ingress-appgw` was enabled by configuring `ingress_application_gateway` for AKS cluster in terraform.

And now we can proceed to configuring our ingresses.

## Ingress spec - single tenant
For our first tenant, living on `customer1.example.com`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: 'ingress'
  namespace: 'customer1-ns'
  annotations:
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: 'wildcard_certificate'
    appgw.ingress.kubernetes.io/ssl-redirect: 'true'
spec:
  ingressClassName: 'azure-application-gateway'
  rules:
    - host: 'customer1.example.com'
      http:
        paths:
          - path: '/'
            pathType: 'Prefix'
            backend:
              service:
                name: 'backend'
                port:
                  number: 80
```

The most important line here is this: `ingressClassName: 'azure-application-gateway'`
That's what makes AGIC take rules of this ingress and turn them into AGW rules.

So when AGIC takes control it does following with AGW objects:
- `host` line defines a source host for **listener** to be set up.
- for said listener it defines a **backend pool** of k8s pod IPs
-  `appgw.ingress.kubernetes.io/appgw-ssl-certificate: 'wildcard_certificate'` - use certificate that was added to AGW.  When this is set, the **listener** for above host becomes https and listens 443 port.
	- Name must match an existing certificate, obviously, otherwise AGW entities won't be updated and error will be logged. Remember, we've set it in AGW terraform deployment, in `ssl_certificate` block.
- `appgw.ingress.kubernetes.io/ssl-redirect: 'true'` forces http traffic to be redirected to https above. This will create one more **listener**.

And that's all! Of course, we can modify paths too and other objects.  Also there are more annotations, documented here: [Application Gateway Ingress Controller annotations | Microsoft Learn](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-annotations)

## Ingress spec - multiple tenants

But what of our second tenant? As I've stated in the beginning - we'll be using a separate ingress in this tenant's namespace.

So, only 2 lines are changed - `namespace` and `host` (certificate can be modified too, of course, but in our case it's wildcard for `*.example.com`)
```yaml
...
metadata:
  namespace: 'customer2-ns'
...
spec:
  ...
  rules:
    - host: 'customer2.example.com'
    ...
```

## Helm

And since we're using Helm to deploy, specifying the namespace can be omitted (it is set at deployment level).

We pass the domain name in `spec.rules`  section, and additionally we parametrize the SSL certificate name in `annotations`, in case we need different ones for different tenants in the future. Depending on values schema, this can look like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: 'ingress'
  annotations:
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: {{ .Values.ingress.certificateName }}
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: 'azure-application-gateway'
  rules:
    - host: {{ .Values.ingress.domain }}
      http:
        paths:
          - path: '/'
            pathType: 'Prefix'
            backend:
              service:
                name: 'backend'
                port:
                  number: 80
```


# Conclusion

That's how multi-tenant ingresses can be created and managed in AKS with AGW used as ingress. While this can look complicated, it is not - most complications are related to terraform (like AGW default rules to ignore) and security that's related (key vault connection, but this can be actually avoided).

There is much to try and experiment with, for example, add more than one path to ingress rules. In our example case, the application is fairly simple, with only one endpoint for clients, hiding all the logic inside - no more than one rule is needed.