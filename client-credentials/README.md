# Deploying the Authorization Code (Authcode) Sample Application

The Authorization Code OAuth2 grant type is the most commonly used for web applications deployed into Cloud Foundry.

This sample application integrates with the [UAA](https://github.com/cloudfoundry/uaa) using the [authorization code](https://tools.ietf.org/html/rfc6749#section-4.1) 
OAuth2 grant type. This sample application relies on the [Pivotal Single Sign-On Service](https://docs.pivotal.io/p-identity/1-9/index.html) 
to automatically register this sample application as an OAuth2 client of the UAA and the 
[SSO CFEnv Processor](https://github.com/pivotal-cf/java-cfenv/tree/master/java-cfenv-boot-pivotal-sso) to automatically consume those configurations.

App-specific OAuth2 client configurations are made using the environment variables section of the sample app's [`manifest.yml`](./manifest.yml) 
file.

## Prerequisites:

1. An operator must have installed the [Pivotal Single Sign-On Service](https://docs.pivotal.io/p-identity/1-9/index.html)
1. An operator must have [configured at least one plan](https://docs.pivotal.io/p-identity/1-9/manage-service-plans.html) for the SSO Service that is visible to your Org.
1. The person using this sample app must know login credentials for a user in this plan. For new plans, an operator may need to [create a user](http://docs.pivotal.io/p-identity/1-9/manage-users.html).

### Step 0: Deploy a sample resource server

The goal of applications obtaining tokens using the OAuth2 authcode grant type is to be able to use those tokens to perform privileged 
actions on another service fulfilling the role of a resource server.  This sample authcode application is meant to obtain tokens for use with
the [sample resource server application](../resource-server) which implements a simple TODO application.

As a result of this relationship between the authcode client application and the resource server, having pushed a sample resource server 
app is a required prerequesite for working through the rest of this tutorial. [Follow these instructions](../resource-server/README.md) to 
deploy a sample resource server if you have not already done so.

### Step 1: Create an identity service instance

Using the CF CLI, login and target the space where you'd like the sample app to reside.

Using the plan created as part of the Prerequisites, create a service instance in your space

    cf create-service p-identity <plan-name> <service-instance-name>
    
The name of your service instance can be whatever you like.

### Step 2: Update authcode manifest.yml with the location of the sample resource server

The [`manifest.yml`](./manifest.yml) includes [a configuration block](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest.html#env-block) 
called `env`. This section is used to list environment variables that will be available to the deployed application.

In the `env` section of [`manifest.yml`](./manifest.yml), you must update the value of `RESOURCE_URL` with the location of your deployed 
resource server application. Replace `RESOURCE_URL: https://resource-server-sample.<your-domain>.com` with a real url.

NOTE: You must leave off the trailing slash (`/`) in the `RESOURCE_URL`.

### Step 3: Update authcode manifest.yml with the name of your identity service instance

The [`manifest.yml`](./manifest.yml) includes [a configuration block](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest.html#services-block) 
called `services`. Your app will be bound to any service instances you list in this section when it is pushed.

Make sure that the `services` block includes the name of the service instance created in Step 1.

### Step 4: Deploy Sample Application to Pivotal Cloud Foundry
    
Build the jar for our sample application:

    ./gradlew clean build
    
This should result in the creation of an artifact `build/libs/authcode.jar`. Next push the authcode sample app:

    cf push --random-route

Running `cf push` should result in
 
  - The app being bound to the identity service instance, which results in the creation of a new client registration for the sample app in the UAA.
  - The OAuth client id and client secret from the UAA are provided to your application through the `VCAP_SERVICES` environment variable. You can view these values yourself with `cf env authcode-sample`.
  - When the app starts, the SSO CFEnv Processor reads `VCAP_SERVICES` and translates configuration from `p-identity` into the configuration needed by `org.springframework.security.oauth` to make the sample application OAuth-aware.

You can verify the app is successfully running by viewing the output of `cf apps`. You can visit the authcode app by navigating to `https://<authcode-app-url>` where `authcode-app-url` is the route output from `cf push` or `cf apps`.

## Testing the TODO application

You can test this sample application with users who have the `todo.read` and `todo.write` scopes for your plan. An operator can create these
users with these permissions using the steps [here](https://docs.pivotal.io/p-identity/configure-id-providers.html#add-to-int).

To create the resource and permissions, an operator must follow [these steps](http://docs.pivotal.io/p-identity/manage-resources.html). After 
the resource and permissions have been created, you need to update the authcode-sample app with the previously created scopes on the App dashboard.

# Bootstrap Application Client Configurations for the Pivotal Single Sign-On Service Instance
Beginning in SSO 1.4.0, you can use the following values your application's manifest to bootstrap client configurations for your applications automatically when binding or rebinding your application to the service instance. These values will be automatically populated to the client configurations for your application through CF environment variables.

When you specify your own scopes and authorities, consider including openid for scopes on auth code, implicit, and password grant type applications, and uaa.resource for client credentials grant type applications, as these will not be provided if they are not specified.

The table below provides a description and the default values. Further details and examples are provided in the sample application manifests.

| Property Name | Description | Default |
| ------------- | ------------- | ------------- |
| name | Name of the application | (N/A - Required Value) |
| GRANT_TYPE | Allowed grant type for the application through the SSO service - only one grant type per application is supported by SSO | authorization_code |
| SSO_IDENTITY_PROVIDERS | Allowed identity providers for the application through the SSO service plan | uaa |
| SSO_REDIRECT_URIS | Comma separated whitelist of redirection URIs allowed for the application - Each value must start with http:// or https:// |  (Will always include the application route) |
| SSO_SCOPES | Comma separated list of scopes that belong to the application and are registered as client scopes with the SSO service. This value is ignored for client credential grant type applications. |  openid |
| SSO_AUTO_APPROVED_SCOPES | Comma separated list of scopes that the application is automatically authorized when acting on behalf of users through SSO service | <Defaults to existing scopes/authorities> |
| SSO_AUTHORITIES | Comma separated list of authorities that belong to the application and are registered as client authorities with the SSO service. Authorities are restricted to the space they were originally created. Privileged identity zone/plan administrator scopes (e.g. scim.read, idps.write) cannot be bootstrapped and must be assigned by zone/plan administrators. This value is ignored for any grant type other than client credentials. | uaa.resource |
| SSO_REQUIRED_USER_GROUPS | Comma separated list of groups a user must have in order to authenticate successfully for the application | (No value) |
| SSO_ACCESS_TOKEN_LIFETIME | Lifetime in seconds for the access token issued to the application by the SSO service | 43200 |
| SSO_REFRESH_TOKEN_LIFETIME | Lifetime in seconds for the refresh token issued to the application by the SSO service | 2592000 (not used for client credentials) |
| SSO_RESOURCES |  Resources that the application will use as scopes/authorities for the SSO service to be created during bootstrapping if they do not already exist - The input format can be referenced in the provided sample manifest. Note that currently all permissions within the same top level permission (e.g. todo.read, todo.write) must be specified in the same application manifest. Currently you cannot specify additional permissions in the same top level permission (e.g. todo.admin) in additional application manifests.| (No value) |
| SSO_ICON |  Application icon that will be displayed next to the application name on the Pivotal Account dashboard if show on home page is enabled - do not exceed 64kb | (No value) |
| SSO_LAUNCH_URL |  Application launch URL that will be used for the application on the Pivotal Account dashboard if show on home page is enabled | (Application route) |
| SSO_SHOW_ON_HOME_PAGE |  If set to true, the application will appear on the Pivotal Account dashboard with the corresponding icon and launch URL| True |

To remove any variables set through bootstrapping, you must use `cf unset-env <APP_NAME> <PROPERTY_NAME>` and rebind the application.

## Troubleshooting

#### Scenario:

You have received an error during `cf push`, or an explict `cf bind-service` call, to the SSO Service that looks like this:

```
Binding Failure:
Creating app client-credentials...
Mapping routes...
Binding services...
Unexpected Response
Response code: 502
CC code:       0
CC error code:
Request ID:    78c53a54-8e03-4922-4833-8b51156e6078::fd0c88e1-8e2e-4eb3-85db-cfea8e078e01
Description:   {
  "description": "Service broker error: The resource name \"todo\" already exists in another space. Please enter a unique resource name before saving",
  "error_code": "CF-ServiceBrokerBadResponse",
  "code": 10001,
  "http": {
    "uri": "https://p-identity-broker.<CF_DOMAIN>/v2/service_instances/bc9a7563-e8b0-4dce-aef2-d7d50b486d9d/service_bindings/80adcbdf-7832-4d45-a58f-76fde5056c8f",
    "method": "PUT",
    "status": 500
  }
}
```

This means that the `todo` resource is a cross-space resource, namely, a resource that has already been created on the same SSO service plan in a different space. Since SSO enforces some extra restrictions on using cross-space resource, you have to perform some additional steps as a workaround. For more information see the [SSO Documentation](https://docs.pivotal.io/p-identity/manage-resources.html#space-protection).

#### Solution:

1. Comment out the `SSO_RESOURCES` in the `manifest.yml`

1. Unset the `SSO_RESOURCES` env var on the deployed `client-credentials` application.
     ```
     cf unset-env client-credentials SSO_RESOURCES
     ```
1. Push the `client-credentials` application again.
     ```
     cf push
     ```   
     
Removing the `SSO_RESOURCES` env var will result in a call to bind the application to the SSO Service Instance without an attempt to create any additional Groups in UAA. However, the SSO Client created for your application will be allowed to use these Groups and Scopes.

** Important Authorization Followup **

For client-credentials applications, there will be some additional work to allow your SSO Application access to the proper cross-space `authorities` which will result in the `todo.read` and `todo.wrote` scopes on your token. Client Crednetials `authorities` are handled a bit differently than User permissions that are used to authorize token scopes in the Authorization Code Flow. Since there is no intersection of User permissions involved in the Client Credentials grant type token scopes, you will need SSO Plan Admin intervention to ultimately receive a token with the appropriate cross-space scopes. We have attempted to outline some suggestions on how to correctly authorize your SSO Client Credentials application that was created as a result of this sample app binding to an SSO service.

You may choose one of the follow remediation steps:

* You will need to contact your SSO Plan Administrator to whitelist the Authorites for your SSO Application created from the binding of client-credentials. CF System Operator with access to OpsManager can manage and identify SSO Plan Admistrators through the [SSO Service Plans UI](https://docs.pivotal.io/p-identity/manage-service-plans.html#create-svc-plan). An SSO Plan Administrator will then need to navigate to the [SSO Dev Dashboard](https://docs.pivotal.io/p-identity/manage-service-instances.html#access-svc-instance-developer-dashboard), navigate to the SSO Application with the name `client-credentails` and then check the `todo.read` and `todo.write` checkboxes.
* Ask a CF System Operator with access to OpsManager to create a new SSO Service Plan that you can reference in a `create-service` command to give you a clean namespace for resources.