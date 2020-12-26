<p align="center">
    <a href="https://tryretool.com/"><img src="http://tryretool.com/logo.png" alt="Retool logo" height="100"></a> <br>
<b>Retool lets you build custom internal tools in minutes.</b></p> <br>

# Deploying Retool on-premise

Deploying Retool on-premise ensures that all access to internal data is managed within your own cloud environment. You also have the flexibility to control how Retool is setup within your infrastructure, configure logging, and enable custom SAML SSO using providers like Okta and Active Directory.

# Table of contents
- [Simple deployments](#simple-deployments)
    - [EC2 and Docker](#deploying-on-ec2)
    - [Heroku](#deploying-on-heroku)
    - [Aptible](#running-retool-using-aptible)
    - [Render](#deploying-to-render)
- [Managed deployments](#managed-deployments)
    - [ECS](#deploying-on-ecs)
    - [ECS + Fargate](#deploying-on-ecs-with-fargate)
    - [Kubernetes](#deploying-on-kubernetes)
    - [Kubernetes + Helm](#deploying-on-kubernetes-with-helm)
- [Login and SSO](#login-and-sso)
    - [Adding Google login](#adding-google-login)
    - [Enabling Okta SAML SSO](#enabling-okta-saml-sso)
    - [Disabling password-based sign-in](#disabling-password-based-sign-in)
- [Additional features](#additional-features)
    - [Health check endpoint](#health-check-endpoint)
    - [Environment variables](#environment-variables)
- [Troubleshooting](#troubleshooting)
- [Docker cheatsheet](#docker-cheatsheet)


## Simple Deployments

Get set up in 15 minutes by deploying Retool on a single machine. 

### Deploying on EC2
Spin up a new EC2 instance. If using AWS, use the following steps:
1. Click **Launch Instance** from the EC2 dashboard.
1. Click **Select** for an instance of Ubuntu `16.04` or higher.
1. Select an instance type of at least `t2.medium` and click **Next**.
1. Ensure you select the VPC that also includes the databases / API’s you will want to connect to and click **Next**.
1. Increase the storage size to `60` GB or higher and click **Next**. 
1. Optionally add some Tags (e.g. `app = retool`) and click **Next**. This makes it easier to find if you have a lot of instances.
1. Set the network security groups for ports `80`, `443`, `22` and `3000`, with sources set to `0.0.0.0/0` and `::/0`, and click **Review and Launch**. We need to open ports `80` (http) and `443` (https) so you can connect to the server from a browser, as well as port `22` (ssh) so that you can ssh into the instance to configure it and run Retool. By default on a vanilla EC2, Retool will run on port `3000`. 
1. On the **Review Instance Launch** screen, click **Launch** to start your instance.
1. If you're connecting to internal databases, whitelist the VPS's IP address in your database.
1. From your command line tool, SSH into your EC2 instance.
1. Run the command `git clone https://github.com/tryretool/retool-onpremise.git`.
1. Run the command `cd retool-onpremise` to enter the cloned repository's directory.
1. Run `./install.sh` to install Docker and Docker Compose.
1. In your `docker.env` (file only created after running `./install.sh`) add the following:
    ```
    # License key granted to you by Retool
    LICENSE_KEY=YOUR_LICENSE_KEY 

    # This is necessary if you plan on logging in before setting up https
    COOKIE_INSECURE=true 
    ```
1. Run `docker-compose up -d` to start the Retool server.
1. Run `sudo docker-compose ps` to make sure all the containers are up and running.
1. Navigate to your server's IP address in a web browser. Retool should now be running on port `3000`.
1. Click Sign Up, since we're starting from a clean slate. The first user to into an instance becomes the administrator. 

### Deploying on Heroku

Just use the Deploy to Heroku button below!

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

#### Updating a Heroku deployment

To update a Heroku deployment that was created with the button above, you may first set up a `git` repo to push to Heroku

```
$ heroku login
$ git clone https://github.com/tryretool/retool-onpremise
$ cd retool-onpremise
$ heroku git:remote -a YOUR_HEROKU_APP_NAME
```

To update Retool (this will automatically fetch the latest version of Retool)

```
$ git commit --allow-empty -m 'Redeploying'
$ git push heroku master
```

### Manually setting up Retool on Heroku

Alternatively, you may follow the following steps to deploy to Heroku

1. Install the Heroku CLI, and login. Documentation for this can be found here: https://devcenter.heroku.com/articles/getting-started-with-nodejs#set-up
1. Clone this repo `git clone https://github.com/tryretool/retool-onpremise`
1. Change the working directory to the newly cloned repository `cd ./retool-onpremise`
1. Create a new Heroku app with the stack set to `container` with `heroku create your-app-name --stack=container`
1. Add a free database: `heroku addons:create heroku-postgresql:hobby-dev`
1. In the `Settings` page of your Heroku app, add the following environment variables:
    1. `NODE_ENV` - set to `production`
    1. `HEROKU_HOSTED` set to `true`
    1. `JWT_SECRET`  - set to a long secure random string used to sign JSON Web Tokens
    1. `ENCRYPTION_KEY` - a long secure random string used to encrypt database credentials
1. Push the code: `git push heroku master`

To lockdown the version of Retool used, just edit the first line under `./heroku/Dockerfile` to:
```
FROM tryretool/backend:X.XX.X
```

### Running Retool using Aptible

1. Add your public SSH key to your Aptible account through the Aptible dashboard
1. Install the Aptible CLI, and login. Documentation for this can be found here: https://www.aptible.com/documentation/deploy/cli.html
1. Clone this repo `git clone https://github.com/tryretool/retool-onpremise`
1. Change the working directory to the newly cloned repository `cd ./retool-onpremise`
1. Create a new Aptible app with `aptible apps:create your-app-name`
1. Add a database: `aptible db:create your-database-name --type postgresql`
1. Set your config variables (your database connection string will be in your Aptible Dashboard and you can parse out the individual values by following [these instructions](https://www.aptible.com/documentation/deploy/reference/databases/credentials.html#using-database-credentials)). Be sure to rename `EXPIRED-LICENSE-KEY-TRIAL` to the license key provided to you. 
    ```
    aptible config:set --app your-app-name \
        POSTGRES_DB=your-db \
        POSTGRES_HOST=your-db-host \
        POSTGRES_USER=your-user \
        POSTGRES_PASSWORD=your-db-password \
        POSTGRES_PORT=your-db-port \
        POSTGRES_SSL_ENABLED=true \
        FORCE_SSL=true \
        NODE_ENV=production \
        JWT_SECRET=$(cat /dev/urandom | base64 | head -c 256) \
        ENCRYPTION_KEY=$(cat /dev/urandom | base64 | head -c 64) \
        LICENSE_KEY=EXPIRED-LICENSE-KEY-TRIAL
    ```
1. Set your git remote which you can find in the Aptible dashboard: `git remote add aptible your-git-url`
1. Push the code: `git push aptible master`
1. Create a default Aptible endpoint
1. Navigate to your endpoint and sign up as a new user in your Retool instance

### Deploying to Render

Just use the Deploy to Render button below!

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/render-examples/retool)

## Managed deployments

Deploy Retool on a managed service. We've provided some starter template files for Cloudformation setups (ECS + Fargate), Kubernetes, and Helm. 

### Deploying on ECS

We provide a [template file](/cloudformation/retool.yaml) for you to get started deploying on ECS. 

1. In the ECS Dashboard, click **Create Cluster**
2. Select `EC2 Linux + Networking` as the cluster template.
3. Select `t2.medium` as the instance type.

![Instance configuration](https://files.readme.io/346e95b-Screen_Shot_2020-12-24_at_6.20.57_PM.png)

4. Select the VPC in which you’d like to launch the ECS cluster; make sure that you select a [public subnet](https://stackoverflow.com/questions/48830793/aws-vpc-identify-private-and-public-subnet).
5. Download the [retool.yaml](/cloudformation/retool.yaml) file, and add your license key and other relevant variables.
5. Go to the AWS Cloudformation dashboard, and click **Create Stack with new resources → Upload a template file**. Upload your edited `retool.yaml` file.
6. Then, enter the following parameters:
    - Cluster: the name of the ECS cluster you created earlier
    - DesiredCount: 2
    - Environment: staging
    - Force: false
    - Image: `tryretool/backend:latest`
    - MaximumPercent: 250
    - MinimumPercent: 50
    - SubnetId: Select 2 subnets in your VPC - make sure these subnets are public (have an internet gateway in their route table)
    - VPC ID: select the VPC you want to use 
7. Click through to create the stack; this could take up to 15 minutes; you can monitor the progress of the stack being created in the `Events` tab in Cloudformation
8. After everything is complete, you should see all the resources with a `CREATE_COMPLETE` status.
9. In the **Outputs** section within the CloudFormation dashboard, you should be able to find the ALB DNS URL. This is where Retool should be running.

### Deploying on ECS with Fargate

We provide Fargate template files supporting [public](/cloudformation/fargate.yaml) and [private](/cloudformation/fargate.private.yaml) subnets. 

1. In the ECS Dashboard, click **Create Cluster**
1. In **Step 1: Select a cluster template**, select `Networking Only (Powered by AWS Fargate)` as the cluster template.
1. In **Step 2: Configure cluster**, be sure to enable CloudWatch Container Insights. This will help us monitor logs and the health of our deployment through CloudWatch.
1. Download the [public](/cloudformation/fargate.yaml) or [private](/cloudformation/fargate.private.yaml) template file, and add your license key and other relevant variables.
1. Go to the AWS Cloudformation dashboard, and click **Create Stack with new resources → Upload a template file**. Upload your edited `.yaml` file.
1. Enter the following parameters:
    - Cluster: the name of the ECS cluster you created earlier
    - DesiredCount: 2
    - Environment: staging
    - Force: false
    - Image: `tryretool/backend:latest`
    - MaximumPercent: 250
    - MinimumPercent: 50
    - SubnetId: Select 2 subnets in your VPC - make sure these subnets are public (have an internet gateway in their route table)
    - VPC ID: select the VPC you want to use  
1. Click through to create the stack; this could take up to 15 minutes; you can monitor the progress of the stack being created in the `Events` tab in Cloudformation
1. In the **Outputs** section, you should be able to find the ALB DNS URL.
1. Currently the load balancer is listening on port 3000; to make it available on port 80 we have to go to the **EC2 dashboard → Load Balancers → Listeners** and click Edit to to change the port to 80.
    - If you get an error that your security group does not allow traffic on this listener port, you must add an inbound rule allowing HTTP on port 80.
1. In the **Outputs** section within the CloudFormation dashboard, you should be able to find the ALB DNS URL. This is where Retool should be running.

### Deploying on Kubernetes

1. Navigate into the `kubernetes` directory
2. Copy the `retool-secrets.template.yaml` file to `retool-secrets.yaml` and inside the `{{ ... }}` sections, replace with a suitable base64 encoded string. 
    1. To base64 encode your license key, run `echo -n <license key> | base64` in the command line. Be sure to add the `-n` character, as it removes the trailing newline character from the encoding.
    1. If you do not wish to add google authentication, replace the templates with an empty string.
    1. You will need a license key in order to proceed.
3. Run `kubectl apply -f ./retool-secrets.yaml`
4. Run `kubectl apply -f ./retool-postgres.yaml`
4. Run `kubectl apply -f ./retool-container.yaml`

For ease of use, this will create a postgres container with a persistent volume for the storage of Retool data. We recommend that you use a managed database service like RDS as a long-term solution. The application will be exposed on a public ip address on port 3000 - we leave it to the user to handle DNS and SSL.

Please note that by default Retool is configured to use Secure Cookies - that means that you will be unable to login unless https has been correctly setup.

To force Retool to send the auth cookies over HTTP, please set the `COOKIE_INSECURE` environment variable to `'true'` in `./retool-container.yaml`. Do this by adding the following two lines to the `env` section.

```
        - name: COOKIE_INSECURE
          value: 'true'
```

Then, to update the running deployment, run `$ kubectl apply -f ./retool-container.yaml`

#### Updating Retool on Kubernetes
To update Retool on Kubernetes, you can use the following command:

```
$ kubectl set image deploy/api api=tryretool/backend:X.XX.X
```

The list of available version numbers for X.XX.X are available here: https://updates.tryretool.com/


### Deploying on Kubernetes with Helm

1. A helm chart is included in this repository under the ./helm directory
2. The available parameters are documented using comments in the ./helm/values.yaml file

```
helm install ./helm
```

3. Persistent volumes are not reliable - we recommend that a long-term installation of Retool host the database on an externally managed database like RDS. Please disable the included `postgresql` chart by setting `postgresql.enabled` to `false` and then specifying your external database through the `config.postgresql.*` properties.

## Login and SSO

### Adding Google Login

Create a Google oauth client using the tutorial below:

https://developers.google.com/identity/sign-in/web/devconsole-project

The Oauth Client screen should end up looking something like this:
![Sample Google Oauth Config Screen](https://i.imgur.com/yOoYGQd.png)

Add the following to your `docker.env` file

```
CLIENT_ID={YOUR_GOOGLE_CLIENT_ID}
CLIENT_SECRET={YOUR_GOOGLE_CLIENT_SECRET}
RESTRICTED_DOMAIN=yourcompany.com
```

Restart the server and you will have Google login for your org!

In Kubernetes, instead of editing the `docker.env` file, place the base64 encoded version of these strings inside the kubernetes secrets file

### Enabling Okta SAML SSO

Below is a guide to integrating Retool with Okta.

1. Login into Okta as an admin. Make sure to use the Classic UI.
1. Navigate to the `Applications` section of Okta
1. Create a new application using the SAML wizard. More information can be found here: https://help.okta.com/en/prod/Content/Topics/Apps/Apps_App_Integration_Wizard.htm
1. When presented with the `Create a New Appication Integration` dialog, choose:
    1. Platform: `Web`
    1. Sign on method: `SAML 2.0`
1. You will then be directed to a 3 step wizard.
    1. For the first step (General Settings) , name the application 'Retool', and then press next.
    1. For the second step (Configure SAML):
        1. Single sign on URL: https://retool.yourcompany.com/saml/login
        1. Audience URI: retool.yourcompany.com
        1. Attribute Statements: Retool requires three attributes to be present:
            1. firstName: user.firstName
            1. lastName: user.lastName
            1. email: user.email
        1. Leave everything else as default. See below for an example of a valid configuration.
          ![Sample Okta SAML Configuration](https://i.imgur.com/K7VQSiBg.png)
    1. For the third step 
        1. Check 'I'm an Okta customer adding an internal app"
        1. Check "This is an internal app that we have created"
        1. Click 'Finish'
1. The last step will be to provide the Retool instance with the Identity Provider metadata. To do this, press the 'View Setup Instructions' button. A new page with IDP metadata will appear. Copy the XML at the bottom of the page, and then:
    1. If you are using the `docker-compose` setup, add the following line to your `docker.env` file. The XML content will contain multiple lines, so make sure to remove all line endings so that the entire XML string is one line in the `docker.env file`
        ```
        SAML_IDP_METADATA=<?xml version="1.0" encoding="UTF-8"?>...</md:EntityDescriptor>
        ```
    1. If you are using Heroku to deploy Retool, add a new environment variable called `SAML_IDP_METADATA` with the value of the XML document. No further steps are required.

### Disabling password-based sign-in

To disable the use of email + password sign-in, define the `RESTRICTED_DOMAIN` environment variable. For example, using the `docker.env` file, use this:
```
RESTRICTED_DOMAIN=yourcompany.com
```

Note that, if you are using this in conjuction with Google login, Retool will compare the value supplied to `RESTRICTED_DOMAIN` with the domain of users that attempt to authenticate with Google SSO and reject accounts from different domains.

## Additional features

### Environment Variables

You can set environment variables to enable custom functionality like [managing secrets](https://docs.retool.com/docs/secret-management-using-environment-variables), customizing logs, and much more. For a list of all environment variables visit our [docs](https://docs.retool.com/docs/environment-variables).

### Health check endpoint 

Retool also has a health check endpoint that you can set up to monitor liveliness of Retool. You can configure your probe to make a `GET` request to `/api/checkHealth`.

## Troubleshooting

- On Kubernetes, I get the error `SequelizeConnectionError: password authentication failed for user "..."`
    - Make sure that the secrets that you encoded in base64 don't have trailing whitespace! You can use `kubectl exec printenv` to help debug this issue.
    - Run `echo -n '<license key> | base64` in the command line. The `-n` character removes the trailing newline character from the encoding.
- I can't seem to login?
    - If you have not enabled SSL yet, you will need to add the line `COOKIE_INSECURE=true` to your `docker.env` file / environment configuration so that the authentication cookies can be sent over http. Make sure to run `sudo docker-compose up -d` after modifying the `docker.env` file.
- `TypeError: Cannot read property 'licenseVerification' of null` or `TypeError: Cannot read property 'name' of null`
    - There is an issue with your license key. Double check that the license key is correct and that it has no trailing whitespaces. 


## Docker cheatsheet

Below is a cheatsheet for useful Docker commands. Note that you may need to prefix them with `sudo`. 

| Command                     | Description                                                                                                                     | 
| ----------------------------|-------------------------------------------------------------------------------------------------------------------------------| 
| `docker-compose up -d`      | Builds, (re)creates, starts, and attaches to containers for a service. `-d`allows containers to run in background (detached). | 
| `docker-compose down`       | Stops and remove containers and networks                                                                                      |
| `docker-compose stop`       | Stops containers, but does not remove them and their networks                                                                 |
| `docker ps -a`              | Display all Docker containers                                                                                                 |
| `docker-compose ps -a`      | Display all containers related to images declared in the `docker-compose` file. 
| `docker logs -f <container_name>` | Stream container logs to stdout                                                                                     |
| `docker exec -it <container_name> psql -U <postgres_user> -W <postgres_password> <postgres_db>` | Runs `psql` inside a container                            |
| `docker kill $(docker ps -q)` | Kills all running containers                                                                                                |
| `docker rm $(docker ps -a -q)` | Removes all containers and networks                                                                                        |
| `docker rmi -f $(docker images -q)`| Removes (and un-tags) all images from the host                                                                         |
| `docker volume rm $(docker volume ls -q)` | Removes all volumes and completely wipes any persisted data                                                     |
