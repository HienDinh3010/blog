Requirements:

[Post]
- Create a post
- List all post

[Comment]
- Create a comment of a post
- List all comments (belong to a post)

Posts Service:
- (1) POST /posts {title: string}
- (2) GET  /posts

Comments Service
- (3) POST /posts/:id/comments { content: string }
- (4) GET  /posts/:id/comments

Problem: When we reload the page and get the list of posts, for every post we call API get list of comment belong to each post.
It turns out we need to call API (4) multiple times. If we have 3 posts we call API (4) 3 times. Is there any other way to optimize the number of call to Comments Service?

Answer: Using Async Communication in microservices - Event Bus
- Whenever user create post or comment, post service and comment service will emit an event to any services register to listen to this even.
- Create a Query Service which is listening to Create Post and Create Commment action then update its database
    <table>
        <thead>
            <tr>
                <th>id</th>
                <th>title</th>
                <th>comments</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>post id</td>
                <td>post title</td>
                <td>list of comment {id, content}</td>
            </tr>
        </tbody>
    </table>
  <img width="789" alt="image" src="https://github.com/user-attachments/assets/17209670-5d02-41d4-b455-924d9536883d">

Adding new feature: Add in comment moderation. Flag comments that contain the word 'orange'.
Solve by: create moderation service to listen to CommentCreated event, this moderation service will update status of comment (rejected/approved) then emitting the CommentModerated event back to comment service. 
Comment service will listent to CommentModerated event then emiting the CommentUpdated event.
Finally, Query service will listen to CommentUpdated event

Problem: What if query service is die while PostCreated or CommentCreated event happens?
Solve by: store events in event-bus service then when query service is up again, it can handle the unprocess missed event.

Deployment issues:

Your computer:

<table>
    <thead>
        <tr>
            <th>Posts</th>
            <th>Comments</th>
            <th>Query</th>
            <th>Moderation</th>
            <th>Event Bus</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Port 4000</td>
            <td>Port 4001</td>
            <td>Port 4002</td>
            <td>Port 4003</td>
            <td>Port 4005</td>
        </tr>
    </tbody>
</table>

Virtual Machine:

<table>
    <thead>
        <tr>
            <th>Posts</th>
            <th>Comments</th>
            <th>Query</th>
            <th>Moderation</th>
            <th>Event Bus</th>
            <th>Comments</th>
            <th>Comments</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Port 4000</td>
            <td>Port 4001</td>
            <td>Port 4002</td>
            <td>Port 4003</td>
            <td>Port 4005</td>
            <td>Port 4006</td>
            <td>Port 4007</td>
        </tr>
    </tbody>
</table>


Virtual machine + Second Virtual machine:
- Virtual machine contains original services
<table>
    <thead>
        <tr>
            <th>Posts</th>
            <th>Comments</th>
            <th>Query</th>
            <th>Moderation</th>
            <th>Event Bus</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Port 4000</td>
            <td>Port 4001</td>
            <td>Port 4002</td>
            <td>Port 4003</td>
            <td>Port 4005</td>
        </tr>
    </tbody>
</table>

- Second virtual machine contains the duplicated Comments service running in different ports
<table>
    <thead>
        <tr>
            <th>Comments</th>
            <th>Comments</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Port 4006</td>
            <td>Port 4007</td>
        </tr>
    </tbody>
</table>

- Consider a scenario where the number of requests to our application varies throughout the day. For example, at 10 AM, there is a high volume of requests, but by 10 PM, the workload decreases significantly. In such cases, we can optimize resource usage by turning off the second virtual machine at 10 PM.

Docker
- Docker creates and manages containers. A container is an isolated computing environment including everything we need to run an application.
- In our blog application, each service runs within its own container. We can also duplicate the container for a service to scale application.
- Why we need to use docker?
Because (1) Running our app right now makes big assumptions about our environment. (2) Running our app requires precise knowledge of how to start it (npm start) => Docker solves both issues. Containers wrap up everything we need for an application, how to start and run it.

Kubernetes 
- K8s cluster includes list of nodes and a MASTER to manage all these nodes.
- Node a virtual machine that will run our containers.
- Pod is a running container, a pod can run multiple containers.
- Based on config file, K8s will create containers.
- Service provides an URL to access a running container.
Service in K8s is not service in Microservice :)
- Deployment: Monitor a set of pods, make sure they are running and restarts them if they crash.
- K8s makes sure the communication between containers is smooth. We don't care how many duplicated containers of a post service, we only need to tell k8s "Hi, from event bus, I want to call post service". k8s will take care of the rest.
- K8s config files
    + Define the deployments, pods and services (referred to as 'Objects') that we want to create in yaml syntax.
    + We can create Objects without config files (do not do this). We only create objects by running direct commands for testing purpose.

How to updating the image used by a deployment?
- Method 1: Make a change to the code -> Rebuild the image, specify a image version -> In the deployment config file, update the version of image -> Run the command: "kubectl apply -f [depl file name]"
- Method 2: The deployment must be using the 'latest' tag in the pod spec section -> Make an update to your code -> Build the image -> Push the image to docker hub -> Run the command "kubectl rollout restart deployment [dept_name]"

Networking With Services: 
4 types of services:
- Cluster IP: Sets up an URL to access a pod. Only exposes pods in the cluster.
- Node Port: Makes a pod accessible from outside the cluster. (Usually only used for dev purposes)
- Load Balancer: Makes a pod accessible from outside the cluster. This is the right way to expose a pod to the outside world.
- External Name: Redirects an in-cluster request to a CNAME url (Only used in rare case - so don't worry)

Error when access http://localhost:30359/posts
If you're using a local Kubernetes cluster (like Minikube or Docker Desktop), you can try using port forwarding to expose the service on your localhost directly: kubectl port-forward svc/posts-srv 30359:4000
Then try accessing http://localhost:30359/posts.


Install Ingress Controller: https://kubernetes.github.io/ingress-nginx/deploy/#quick-start