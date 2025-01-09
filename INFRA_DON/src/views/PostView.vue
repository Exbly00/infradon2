<script lang="ts">
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

// Activation de pouchdb-find pour permettre la recherche
PouchDB.plugin(PouchDBFind)

interface Props {
  name: string
  content: string
  searchQuery: string
  storage?: PouchDB.Database<{}> // Base de données PouchDB (optionnelle)
  posts?: {
    rows: Array<{
      id: string
      key: string
      value: { rev: string }
      doc: Post // Le type des documents
    }>
    total_rows: number
    offset: number
  }
}

interface Post {
  _id: string
  _rev?: string // Ajouté pour gérer les suppressions/modifications
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
      searchQuery: '',
      posts: undefined
    }
  },
  methods: {
    // Méthode pour afficher le prompt et récupérer le contenu modifié
    promptEdit(content: string): string {
      const newContent = window.prompt('Modifier le contenu du post', content)
      return newContent ?? '' // Retourne une chaîne vide si l'utilisateur annule
    },
    initDatabase() {
      const db = new PouchDB('http://admin:admin@localhost:5984/post')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db

      // Créer un index au démarrage
      this.createIndex()
    },
    async getPosts() {
      const response = await this.storage?.allDocs<Post>({ include_docs: true, attachments: true })

      if (response) {
        this.posts = {
          rows: response.rows.map((row) => ({
            id: row.id,
            key: row.key,
            value: { rev: row.value.rev },
            doc: row.doc as Post
          })),
          total_rows: response.total_rows,
          offset: response.offset
        }
      }
    },
    async createPost(e: Event) {
      e.preventDefault()
      e.stopPropagation()

      const post: Post = {
        _id: Math.random().toString(36).substring(2),
        doc: {
          post_name: this.name,
          post_content: this.content,
          attributes: {
            creation_date: new Date().toISOString()
          }
        }
      }

      await this.storage?.put(post)
      this.getPosts()

      // Réinitialisation du formulaire
      this.name = ''
      this.content = ''
    },
    async deletePost(postId: string, rev: string) {
      await this.storage?.remove(postId, rev)
      this.getPosts()
    },
    async editPost(postId: string, updatedContent: string) {
      const post = await this.storage?.get<Post>(postId)
      if (post) {
        post.doc.post_content = updatedContent
        await this.storage?.put(post)
        this.getPosts()
      }
    },
    async createIndex() {
      await this.storage?.createIndex({
        index: {
          fields: ['doc.post_name']
        }
      })
      console.log('Index created')
    },
    async searchPosts() {
      const response = await this.storage?.find({
        selector: { 'doc.post_name': { $regex: this.searchQuery } }
      })

      if (response) {
        this.posts = {
          rows: response.docs.map((doc) => ({
            id: doc._id,
            key: doc._id,
            value: { rev: doc._rev as string },
            doc: doc as Post
          })),
          total_rows: response.docs.length,
          offset: 0
        }
      }
    },
    syncDatabase() {
      const localDB = new PouchDB('local_db')

      // Vérifie si la base de données distante est correctement initialisée
      if (this.storage) {
        PouchDB.replicate(this.storage, localDB, { live: true, retry: true }).on(
          'change',
          (info) => {
            console.log('Replication info:', info)
          }
        )
      } else {
        console.warn("La base de données distante n'est pas initialisée.")
      }
    }
  },
  mounted() {
    this.initDatabase()
    this.getPosts()
    this.getPosts().catch((err) => {
      console.error('Erreur lors de la récupération des posts:', err)
    })
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

  <h2>Recherche</h2>
  <input type="text" v-model="searchQuery" @input="searchPosts" placeholder="Rechercher par nom" />

  <h1>Posts</h1>

  <div v-if="posts && posts.rows.length">
    <div v-for="post in posts.rows" :key="post.id">
      <a v-if="post.doc?.doc" :href="'/posts/' + post.doc._id">
        {{ post.doc.doc.post_name }}
      </a>
      <p v-if="post.doc?.doc">{{ post.doc.doc.post_content }}</p>
      <div class="buttons">
        <button @click="deletePost(post.id, post.value.rev)">Supprimer</button>
        <button @click="editPost(post.id, promptEdit(post.doc.doc.post_content))">Modifier</button>
      </div>
    </div>
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
  margin-top: 20px;
  background: #33ef82;
  padding: 12px;
  border: none;
  border-radius: 12px;
  cursor: pointer;
}

.buttons {
  display: flex;
  gap: 10px; /* Espace entre les boutons */
}
</style>
