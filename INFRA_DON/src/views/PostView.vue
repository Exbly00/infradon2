<script lang="ts">
import PouchDB from 'pouchdb'

interface Props {
  name: string
  content: string
  storage?: PouchDB.Database<{}>
  posts?: PouchDB.Core.AllDocsResponse<Post>
}

interface Post {
  _id: string
  doc: {
    post_name: string
    post_content: string
    attributes: {
      creation_date: string
    }
  }
}

export default {
  data(): Props {
    return {
      name: '',
      content: '',
      posts: undefined
    }
  },
  methods: {
    initDatabase() {
      const db = new PouchDB('http://admin:admin@localhost:5984/post')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    },
    getPosts() {
      this.storage?.allDocs<Post>({ include_docs: true, attachments: true }).then((response) => {
        this.posts = response
      })
    },
    async createPost(e: Event) {
      // Prevent the browser from reloading the page
      e.preventDefault()
      e.stopPropagation()

      const post: Post = {
        _id: Math.random().toString(36).substring(2),
        doc: {
          post_name: this.name,
          post_content: this.content,
          attributes: {
            creation_date: 'today'
          }
        }
      }

      this.storage?.put(post).then(() => {
        // We wait until the `.put` is done to get the posts in order to have the newly
        // created post.
        this.getPosts()
      })

      // Reset form
      this.name = ''
      this.content = ''
    }
  },
  mounted() {
    this.initDatabase()
    this.getPosts()
  }
}
</script>

<template>
  <h1>Post</h1>

  <form v-on:submit="createPost">
    <label for="name">Name</label>
    <input type="text" id="name" name="name" v-model="name" />

    <label for="content">Content</label>
    <textarea name="content" id="content" rows="5" v-model="content"></textarea>

    <button>Envoyer</button>
  </form>

  <h1>Posts</h1>

  <div v-if="posts" v-for="post in posts.rows">
    <a :href="'/posts/' + post.doc?._id">{{ post.doc?.doc.post_name }}</a>
  </div>
</template>

<style>
form {
  display: flex;
  flex-flow: column nowrap;
  gap: 8px;
}

input,
textarea {
  border: 1px solid #222;
  background: #333;
  padding: 6px 12px;
  border-radius: 12px;
  color: #eee;
}

button {
  margin-top: 12px;
  background: #33ef82;
  padding: 12px;
  border: none;
  border-radius: 12px;
  cursor: pointer;
}
</style>
