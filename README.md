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