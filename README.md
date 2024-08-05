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