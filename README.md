Requirements:

[Post]
- Create a post
- List all post

[Comment]
- Create a comment of a post
- List all comments (belong to a post)

Posts Service:
- POST /posts {title: string}
- GET  /posts

Comments Service
- POST /posts/:id/comments { content: string }
- GET  /posts/:id/comments