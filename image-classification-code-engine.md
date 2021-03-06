---
subcollection: solution-tutorials
copyright:
  years: 2020
lastupdated: "2020-10-19"
lasttested: "2020-09-23"

content-type: tutorial
services: codeengine, containers, cloud-object-storage, visual-recognition
account-plan: paid
completion-time: 2h

---

{:step: data-tutorial-type='step'}
{:java: #java .ph data-hd-programlang='java'}
{:swift: #swift .ph data-hd-programlang='swift'}
{:ios: #ios data-hd-operatingsystem="ios"}
{:android: #android data-hd-operatingsystem="android"}
{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}
{:important: .important}
{:note: .note}
{:beta: .beta}

# Visual recognition with {{site.data.keyword.codeengineshort}}
{: #image_classification_code_engine}
{: toc-content-type="tutorial"}
{: toc-services="codeengine, containers, cloud-object-storage, visual-recognition"}
{: toc-completion-time="2h"}

This tutorial may incur costs. Use the [Cost Estimator](https://{DomainName}/estimator/review) to generate a cost estimate based on your projected usage.
{:tip}

{{site.data.keyword.codeenginefull_notm}} is available as a beta service. Beta runtimes and services might be unstable or change frequently. Be aware of [beta limitations](https://{DomainName}/docs/codeengine?topic=codeengine-limits).
{:beta}

In this tutorial, you will learn about {{site.data.keyword.codeenginefull}} by deploying an image classification (visual recognition) application. You will create a {{site.data.keyword.codeengineshort}} project, select the project and deploy {{site.data.keyword.codeengineshort}} components - applications and jobs to the project. You will learn how to bind {{site.data.keyword.cloud_notm}} services to your {{site.data.keyword.codeengineshort}} components. You will also understand the auto-scaling capability of {{site.data.keyword.codeengineshort}} where instances are scaled up or down (to zero) based on incoming workload.
{:shortdesc}

{{site.data.keyword.codeenginefull_notm}} provides a platform to unify the deployment of all of your container-based applications. Whether those applications are functions, traditional 12-factor apps, batch workloads(run-to-completion) or any other container-based workloads, if they can be bundled into a container image, then {{site.data.keyword.codeengineshort}} can host and manage them for you - all on a Kubernetes-based infrastructure. And {{site.data.keyword.codeengineshort}} does this without the need for you to learn, or even know about, Kubernetes. The {{site.data.keyword.codeengineshort}} experience is designed so that you can focus on writing code and not on the infrastructure needed to host it. It is delivered as a managed service on the cloud and built on open-source projects (Kubernetes, Istio, Knative, Tekton, etc.).

{{site.data.keyword.codeengineshort}} helps developers by hiding many of the complex tasks like configuration, dependency management etc., {{site.data.keyword.codeengineshort}} simplifies container-based management and enables you to concentrate on writing code. It also makes available many of the features of a serverless platform, such as "scale-to-zero".

## Objectives
{: #image-classification-code-engine-objectives}

* Understand {{site.data.keyword.codeenginefull}} and how it simplifies the developer experience.
* Understand how easy it is to deploy and scale an application using {{site.data.keyword.codeengineshort}}.
* Learn the use of jobs to execute run to completion workloads.


<p style="text-align: center;">

  ![Architecture](images/solution54-code-engine/architecture_diagram.png)
</p>

1. Developer creates a {{site.data.keyword.codeengineshort}} project and deploys a frontend and a backend {{site.data.keyword.codeengineshort}} application.
2. Developer connects the frontend (UI) app to the backend by modifying the frontend application to set an environment variable value to point to the backend application's endpoint.
3. Developer provisions the required cloud services and binds them to the backend application and jobs by creating secrets and configmap.
4. User uploads an image(s) via the frontend app that is stored in {{site.data.keyword.cos_short}} through the backend application.
5. User runs a {{site.data.keyword.codeengineshort}} job via the backend to classify the image by pushing the image to {{site.data.keyword.visualrecognitionshort}}. The result is then saved to {{site.data.keyword.cos_short}} and displayed in the frontend app when the user clicks the refresh button.

## Before you begin
{: #image-classification-code-engine-prereqs}

This tutorial requires:
* {{site.data.keyword.cloud_notm}} CLI - This CLI tool will enable you to interact with {{site.data.keyword.cloud_notm}}.
   * code-engine/ce plugin (`code-engine/ce`) - Plugins extend the capabilities of the {{site.data.keyword.cloud_notm}} CLI with commands specific to a service. The {{site.data.keyword.codeengineshort}} plugin will give you access to {{site.data.keyword.codeengineshort}} commands on {{site.data.keyword.cloud_notm}}.

You will find instructions to download and install these tools for your operating environment in the [Getting started with tutorials](/getting-started.md#getting-started-with-tutorials) guide.

**Note:** To avoid the installation of these tools you can use the [{{site.data.keyword.cloud-shell_short}}](https://{DomainName}/shell) from the {{site.data.keyword.cloud_notm}} console.
{:tip}


## Create an {{site.data.keyword.codeenginefull_notm}} project
{: #image-classification-code-engine-create_code_engine_project}
{: step}

In this section, you will create a {{site.data.keyword.codeengineshort}} project. A project is a grouping of the components of your project that are typically meant to go together as part of some overall workload.

Putting components into a single project enables you to manage access control more easily. The components within a project share the same private network, which enables them to talk to each other securely.

1. Navigate to [{{site.data.keyword.codeenginefull_notm}} Overview](https://{DomainName}/codeengine/overview) page.
2. On the left pane, click on **Projects** and then click **Create project**,
   - Select a Location
   - Provide a project name and select a Resource group where you will create your project and also the cloud services required in the later steps. Resource groups are a way for you to organize your account resources into customizable groupings.
   - Click on **Create** and then **Confirm & create**
   - Wait until the project `status` changes to **Active**
3. In a terminal on your machine, ensure you're logged in to the `ibmcloud` CLI.
   ```sh
   ibmcloud login
   ```
   {:pre}
4. You will also need to target the resource group where you created your project.
   ```
   ibmcloud target -g <YOUR_RESOURCE_GROUP_NAME>
   ```
   {:pre}
5. Make the command line tooling point to your project
   ```sh
   ibmcloud code-engine project select --name <PROJECT_NAME>
   ```
   {:pre}

## Deploy the frontend and backend apps as {{site.data.keyword.codeengineshort}} applications
{: #image-classification-code-engine-deploy_app}
{: step}

{{site.data.keyword.codeengineshort}} Applications run your code to serve HTTP requests, autoscale up and back down to zero, and offer traffic routing to multiple revisions. In this section, you will deploy your front-end and back-end applications to {{site.data.keyword.codeengineshort}} under the targeted project. This front-end web application will allow users to upload images, while the backend application will write the image to {{site.data.keyword.cos_full_notm}}.

We've already built images for the two applications and pushed them to the public container registry. You will use these pre-built container images to deploy the respective applications.

### Deploy a frontend application
{: #image-classification-code-engine-4}

1. To deploy a new {{site.data.keyword.codeengineshort}} application, you need to run the following command; providing a service name "frontend" and the pre-built container image as a parameter to `--image` flag.

   ```sh
   ibmcloud code-engine application create --name frontend --image ibmcom/frontend
   ```
   {:pre}

   After running this command, you should see some output with a URL to your application. It should look something like: `https://frontend.b7bb43f4-525e.us-south.codeengine.appdomain.cloud`. Make note of this application URL for the next step. With just these two pieces of data (application name and image name), {{site.data.keyword.codeengineshort}} has deployed your application and will handle all of the complexities of configuring it and managing it for you.

   The application source code used to build the container images - `ibmcom/*` is available in a [GitHub repo](https://github.com/IBM-Cloud/code-engine-image-classification) for your reference.
   {:tip}

2. Copy the URL from the `application create` output and open it in a browser to see an output similar to this
   ![](images/solution54-code-engine/frontend-501.png)

   Run `ibmcloud code-engine application get -n frontend` command to see the details of the application. You should see details like the ID, project information, age of the application, the URL to access the application, a Console URL to access your application configuration, Image, Resource allocation, and various revisions, conditions and runtime for your application. Since you only have one revision, you should see that 100% of the traffic is going to the latest revision. You can also check the number of `Instances` and their status.
   {:tip}

3. For troubleshooting and to check the logs of your application, run the following command by replacing the `<INSTANCE_NAME>` with the **name** of the instance from the `ibmcloud code-engine application get -n frontend` command
   ```
   ibmcloud code-engine application logs --instance <INSTANCE_NAME>
   ```
   {:pre}

Congratulations!! You've just deployed a web application to {{site.data.keyword.codeengineshort}} with a simple command and also without needing to know about the intricacies of Kubernetes such as pods, deployments, services, and ingress.

### Scale the application
{: #image-classification-code-engine-5}

When you created the application with the `application create` command, you only passed in an image to use and a name for your application. While this is the minimum amount of information to deploy an application, there are a number of other knobs you have control over. Among others, you can set the number of requests that can be processed concurrently per instance, the amount of CPU for the instance of the application, the amount of memory set for the instance of the application, the environment variables for the application, the maximum and minimum number of instances that can be used for this application, and the port where the application listens for requests.

Most of these values have a default set if nothing is provided as an option when creating the application. Because we did not provide a value, {{site.data.keyword.codeengineshort}} deployed our application with a default max scale of 10, meaning that it will only scale our application up to 10 instances. The default minimum scale is zero, so that when our application is no longer in use, it will scale itself back down to zero.

1. To check the autoscaling capabilities of {{site.data.keyword.codeengineshort}}, we can use a load generator to generate a load against our service. This load generator will simulate about 300 clients hitting the URL for 30 seconds. Navigate to the [load generator URL](https://load.fun.cloud.ibm.com/) and paste the frontend application URL from the step above.
2. Click on **Generate load** to generate traffic.
3. Run the below command to see the instance(pod) count incrementing as part of the autoscaling.
   ```sh
   ibmcloud code-engine application get -n frontend
   ```
   {:pre}

   By default, the maximum number of requests that can be processed concurrently per instance is `10` leading to autoscaling and this value can be changed using `--concurrency or -cn` flag with `application update` command.
   {:tip}

4. The default for maximum number of instances when an application is created is 10 instances, so you should see that there were 10 instances created. If you didn't want to allow as many instances to be created, you can adjust the max scale to be a lower number. While your serverless application can easily scale up, you may depend on a downstream service such as a SQL DB that can only handle a limited number of connections or another rate limited API. Let's try limiting the number of instances for this frontend application.
    ```sh
    ibmcloud code-engine application update --name frontend --max-scale 5
    ```
    {:pre}
5. Once load generation is stopped, wait for a minute to see the instances terminating, eventually scaling down to zero instances.
6. Again, navigate to the [load generator URL](https://load.fun.cloud.ibm.com/) and paste the frontend application URL from the step above. Run the `ibmcloud code-engine application get -n frontend` command to see the instance count increasing to 5.

    Expected Output:
    ```
    frontend-i4fmh-2-deployment-6996489d7c-4lq4w   2/2     Running       0          59s
    frontend-i4fmh-2-deployment-6996489d7c-5d5nx   1/2     Running       0          8s
    frontend-i4fmh-2-deployment-6996489d7c-697g2   1/2     Running       0          6s
    frontend-i4fmh-2-deployment-6996489d7c-t27d2   1/2     Running       0          8s
    frontend-i4fmh-2-deployment-6996489d7c-vzmdd   1/2     Running       0          9s
    ```

### Deploy a backend application and test the connection
{: #image-classification-code-engine-6}

1. To deploy a new backend application to store your images into {{site.data.keyword.cos_full_notm}}, run this command
   ```sh
   ibmcloud code-engine application create --name backend --image ibmcom/backend --cluster-local
   ```
   {:pre}
   The `--cluster-local` flag will instruct {{site.data.keyword.codeengineshort}} to keep the endpoint for this application private, meaning that it will only be available from within the cluster. This is often used for security purposes. In this case, there is no reason to expose the backend application with a public endpoint, since it will not be accessed from outside of the cluster.
   {:tip}

2. Copy and save the private endpoint (URL) from the output to use it in the next command.

   You can run `ibmcloud code-engine application get -n backend` command to check the status and details of the backend application.
   {:tip}

3. The frontend application uses an environment variable (BACKEND_URL) to know where the backend application is hosted. You now need to update the frontend application to set this value to point to the backend application's endpoint. **Replace** the placeholder `<BACKEND_PRIVATE_URL>` with the value from the previous command.
   ```sh
   ibmcloud code-engine application update --name frontend --env BACKEND_URL=<BACKEND_PRIVATE_URL>
   ```
   {:pre}

   The `--env` flag can appear as many times as you would like if you need to set more than one environment variable. This option could have also been used on the `ibmcloud code-engine application create` command for the frontend application if you knew its value at that time.
   {:tip}

4. Refresh the frontend URL on the browser to test the connection to the backend application. You should see a page with an option to upload an image and also an error message from the backend application as the backend is still not connected with the required {{site.data.keyword.cloud_notm}} services to store and process the image. Clicking on **Upload image** should also show a similar error message.
   ![](images/solution54-code-engine/frontend.png)

## Connect the backend application to {{site.data.keyword.cos_short}} service
{: #image-classification-code-engine-0}
{:connect_cloud_services}
{: step}

In this section, you will provision the required {{site.data.keyword.cos_short}} and {{site.data.keyword.visualrecognitionshort}} services and bind the {{site.data.keyword.cos_short}} service to the backend application. The backend application will store the images into the {{site.data.keyword.cos_short}}, while the {{site.data.keyword.visualrecognitionshort}} will be used later in the tutorial to classify the images.

### Provision {{site.data.keyword.cos_short}} and {{site.data.keyword.visualrecognitionshort}} services
{: #image-classification-code-engine-create_services}

1. Create an instance of [{{site.data.keyword.cos_short}}](https://{DomainName}/catalog/services/cloud-object-storage)
   1. Select the **Lite** plan or the **Standard** plan if you already have an {{site.data.keyword.cos_short}} service instance in your account.
   2. Set **Service name** to **code-engine-cos** and select a resource group where you created the {{site.data.keyword.codeengineshort}} project.
   3. Click on **Create**.
2. Under **Service credentials**, click on **New credential**
   1. Give it a name - `cos-for-code-engine` and select **Writer** as the role
   2. Click **Add**.
3. under **Buckets**, create a **Custom** bucket named `<your-initials>-bucket-code-engine` ,
   1. Select **Cross Region** resiliency
   2. Select a Location near to you
   3. Select a **Standard** storage class for high performance and low latency.
   4. Click **Create bucket**
4. On the left pane under **Endpoints**, Select **Cross Region** resiliency and select a Location near to you.
5. Copy the desired **Public** endpoint to access your bucket and **save** the endpoint for quick reference.
6. Create an instance of [{{site.data.keyword.visualrecognitionshort}}](https://{DomainName}/catalog/services/visual-recognition)
   1. Select a region and select **Standard** plan.
   2. Set **Service name** to **code-engine-vr** and select a resource group where you created the {{site.data.keyword.codeengineshort}} project.
   3. Click on **Create**.
7. Under **Service credentials**, click on **New credential**
   1. Give it a name - `vr-for-code-engine` and select **Writer** as the role.
   2. Click **Add**.

### Bind the {{site.data.keyword.cos_short}} service to the backend application
{: #image-classification-code-engine-9}

Now, you will need to pass in the credentials for the {{site.data.keyword.cos_full_notm}} instance you just created into your backend application. You will do this by binding the {{site.data.keyword.cos_short}} service to your application, which automatically adds credentials for a service to the environment variables of the container for your application or job.

1. Create a binding for {{site.data.keyword.cos_short}} service with a prefix `COS` for ease of use in your application. Creating this binding will give your {{site.data.keyword.codeengineshort}} application access to the service credentials for {{site.data.keyword.cos_full_notm}} so that it can store images in COS.
   ```sh
   ibmcloud code-engine application bind --name backend --service-instance code-engine-cos --service-credential cos-for-code-engine --prefix COS
   ```
   {:pre}

   If you have created the {{site.data.keyword.cos_short}} service instance with a different name, pass your service name with `--service-instance` flag.
   {:tip}

2. You will also need to provide the application with your Bucket name where you want to store the images, as well as your COS endpoint. Define a configmap to hold the bucket name and the endpoint as the information isn't sensitive. ConfigMaps are a Kubernetes object, which allows you to decouple configuration artifacts from image content to keep containerized applications portable. You could create this configmap from a file or from a key value pair -- for now we'll use a key value pair with the `--from-literal` flag.
   ```sh
   ibmcloud code-engine configmap create --name backend-configuration --from-literal=COS_BUCKETNAME=<COS_BUCKET_NAME> --from-literal=COS_ENDPOINT=<COS_ENDPOINT>
   ```
   {:pre}

3. With the configmap defined, you can now update the backend application by asking {{site.data.keyword.codeengineshort}} to set environment variables in the runtime of the application based on the values in the configmap. Update the backend application with the following command
   ```sh
   ibmcloud code-engine application update --name backend --env-from-configmap backend-configuration
   ```
   {:pre}

   To create a secret, you would need to use `--env-from-secret` flag. Both secrets and configmaps are "maps"; so the environment variables set will have a name corresponding to the "key" of each entry in those maps, and the environment variable values will be the value of that "key".
   {:tip}

4. To verify whether the backend application is updated with the binding and configmap. You can run the below command and look for the `Service Bindings` and `Environment Variables` sections in the output
   ```sh
   ibmcloud code-engine application get --name backend
   ```
   {:pre}

## Test the application
{: #image-classification-code-engine-test_the_app}
{: step}

Now that you have the backend application connected to the frontend application and provided all the required credentials, let's test it by uploading images for image classification. To test, you will create a job configuration and use the job configuration to run a job to classify images using {{site.data.keyword.visualrecognitionshort}} service.

Jobs in {{site.data.keyword.codeengineshort}} are meant to run to completion as batch or standalone executables. They are not intended to provide lasting endpoints to access like a {{site.data.keyword.codeengineshort}} application does.

### Create a job configuration
{: #image-classification-code-engine-11}

Jobs, unlike applications which react to incoming HTTP requests, are meant to be used for running container images that contain an executable designed to run one time and then exit. When you create a `job`, you can specify workload configuration information that is used each time the job is run. You can create a job from the console or with the CLI.

This job will read images from {{site.data.keyword.cos_full_notm}}, and then classify them using the {{site.data.keyword.visualrecognitionshort}} Service. It will need to have access to service credentials for both services.

1. On a terminal, run the following command to create a job configuration,
   ```sh
   ibmcloud code-engine job create --name backend-job --image ibmcom/backend-job --env-from-configmap backend-configuration
   ```
   {:pre}

   You can set the version of {{site.data.keyword.visualrecognitionshort}} service using the `--env` flag. For versioning, check this [link](https://{DomainName}/apidocs/visual-recognition/visual-recognition-v3#versioning)
   {:tip}

### Bind the {{site.data.keyword.cloud_notm}} services to job
{: #image-classification-code-engine-12}

1. Let's create a binding for {{site.data.keyword.cos_short}} service with a prefix `COS_JOB` to be used with the jobs in the subsequent steps,
   ```sh
   ibmcloud code-engine job bind --name backend-job --service-instance code-engine-cos --service-credential cos-for-code-engine --prefix COS_JOB
   ```
   {:pre}
1. Similarly, let's bind {{site.data.keyword.visualrecognitionshort}} service with a prefix `VR_JOB` to classify the uploaded images,
   ```sh
   ibmcloud code-engine job bind --name backend-job --service-instance code-engine-vr --service-credential vr-for-code-engine --prefix VR_JOB
   ```
   {:pre}
2. To verify whether the job is updated with the binding and configmap. You can run the below command and look for the `Service Bindings` and `Environment Variables` sections in the output
   ```sh
   ibmcloud code-engine job get --name backend-job
   ```
   {:pre}

### Run the job
{: #image-classification-code-engine-13}

1. Go to the frontend UI and **upload images** for classification.
2. To run a job with the configuration created above, use the `jobrun submit` command,
   ```sh
   ibmcloud code-engine jobrun submit --name backend-jobrun --job backend-job
   ```
   {:pre}

   When you run a job, you can override many of the variables that you set in the job configuration. To check the variables, run `ibmcloud code-engine jobrun submit --help`.
   {:tip}

3. To check the status of the `jobrun`, run the following command
   ```sh
   ibmcloud code-engine jobrun get --name backend-jobrun
   ```
   {:pre}
4. For logs, copy the **instance** name from the output of the above command and pass it to `--instance` flag in the following command. It should look something like `backend-jobrun-1-0`.
   ```sh
   ibmcloud code-engine jobrun logs --instance <JOBRUN_INSTANCE_NAME>
   ```
   {:pre}
5. In the frontend UI, click on the **refresh** button (next to Upload image) to see the results for each of the uploaded images by expanding **Show results** on each image.
6. Upload new images or delete individual image by clicking the **delete** icon, resubmit the **jobrun** with the below command and hit the **refresh** button to see the results.
   ```sh
   ibmcloud code-engine jobrun resubmit --jobrun backend-jobrun
   ```
   {:pre}

## Remove resources
{: #image-classification-code-engine-cleanup}
{: step}

1. With the command below, delete the project to delete all it's components (applications, jobs etc.).
   ```sh
   ibmcloud code-engine project delete --name <PROJECT_NAME>
   ```
   {:pre}
2. Navigate to [Resource List](https://{DomainName}/resources/)
3. Delete the services you have created:
 * [{{site.data.keyword.cos_full}}](https://{DomainName}/catalog/services/cloud-object-storage)
 * [{{site.data.keyword.visualrecognitionfull}}](https://{DomainName}/catalog/services/visual-recognition)

## Related resources
{: #image-classification-code-engine-related_resources}

- [{{site.data.keyword.codeenginefull_notm}} Documentation](/docs/codeengine)
