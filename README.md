# AKS Cluster with ingress-appgw addon - WAF and TLS update

The new [ingress controller add-on for AKS](https://docs.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-new) makes deploying an Application Gateway as a cluster ingress controller very easy - just use the addon and it's deployed for you. This is especially important because it sets the security at deploy time, and thus does not require subscription owner rights. However, the configurability of the Application Gateway when deployed this way is minimal; for example, there's no way to deploy it as with the WAF SKU or with different TLS settings. This can be addressed by deploying the application gateway ahead of time (brownfield deployment), but that requires setting security permissions for the AKS cluster MI and thus requires owner rights.

If you deploy the Application Gateway via the addon, it is easy enough to update the configuration after the fact via azure-cli, Az PowerShell, or the portal, but it's difficult to do via ARM templates because the Application Gateway configuration is generated by the addon and being constantly updated by the ingress-appgw-controller component in the AKS cluster. Pushing the configuration via a standard ARM template won't account for these dynamic configuration objects and thus will destroy the configuration from the AKS cluster.

The templates in this repository make use of nesting and referencing to dynamically update the configuration on the AKS cluster Application Gateway in pure ARM template language; no shell scripting or other tools are required.

The template makes use of four [linked templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates#linked-template) to allow us to pass values back and forth properly while insuring resource consistency. The flow for the deployment is as follows:

To begin, [mainTemplate.json](mainTemplate.json) is deployed with the relevant parameters specified. This deployment is targeted on the resource group where the AKS cluster object should end up. This template contains 3 linked templates which deploy in order:

1. [nested-01-deployAKSClusterWithAppGw.json](nested-01-deployAKSClusterWithAppGw.json) begins the process by deploying the actual AKS cluster and Application Gateway. After these deployments are complete, the template returns (via outputs) the resource ID of the Application Gateway and the AKS node resource group (where the application gateway is deployed).

2. [nested-02-getAppGwDetails.json](nested-02-getAppGwDetails.json) is a no-op template - it takes the resource ID of the application gateway from the step 1 outputs and returns two values: the name of the application gateway (from splitting the resource ID on / and returning the last element) and the contents of the application gateway properties. This is done because the [`reference()`](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#reference) function, which is used to return these values, [can only be used in the parameters or outputs section](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#valid-uses-1:~:text=The%20reference%20function%20can%20only%20be,section%20of%20a%20template%20or%20deployment). Once these values have been returned as outputs from the linked template, they can then be passed as inputs to other templates.

3. [nested-03-createUpdateDeployment.json](nested-03-createUpdateDeployment.json) is only used to create a new deployment for step 4. Because the Application Gateway is being created in the AKS cluster node resource group, we have to target the final deployment to that resource group. However, we can't use `reference()` (which is required to get the node resource group name from the step 1 outputs) in the `resourceGroup` field. In order to accomplish the required deployment, the resource group name is passed as a parameter to this template, which then creates a fourth deployment in the node resource group.

4. [nested-04-updateAppGwConfig.json](nested-04-updateAppGwConfig.json) is the template that actually updates the configuration on the Application Gateway. The current state of the object is passed in to the template in the `appGwProperties` parameter. The desired changes are stored in this template in the `changedProperties` variable. The [`union()`](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-object#union) function is used to take the original properties and merge in the changes; any key that exists in `changedProperties` will be added or overwritten, but all other existing configuration from `appGwProperties` will be preserved.