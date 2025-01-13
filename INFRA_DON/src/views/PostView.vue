<script lang="ts">
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

// Définir les types pour les props et les données
interface Props {
  name: string
  content: string
  searchQuery: string
  mediaFile: File | null
  storage?: PouchDB.Database<Post> // Utilisation du type spécifique Post pour la base de données
  posts?: {
    rows: Array<{
      id: string
      key: string
      value: { rev: string }
      doc: Post
    }>
    total_rows: number
    offset: number
  }
}

// Définir l'interface pour les posts
interface Post {
  _id: string
  post_name: string
  post_content: string
  attributes: {
    creation_date: string
  }
  media: any[]
}

// Exporter le composant
export default {
  data(): Props {
    return {
      name: '',
      content: '',
      searchQuery: '',
      mediaFile: null,
      posts: undefined
    }
  },
  methods: {
    // méthode handleFileChange pour gérer le changement de fichier
    handleFileChange(event: Event) {
      const target = event.target as HTMLInputElement | null
      if (target && target.files) {
        this.mediaFile = target.files[0]
      }
    },
    // méthode promptEdit pour afficher une boîte de dialogue de modification
    promptEdit(content: string): string {
      const newContent = window.prompt('Modifier le contenu du post', content)
      return newContent ?? ''
    },
    //méthode initDatabase pour initialiser la base de données
    initDatabase() {
      // Base distante
      const remoteDB = new PouchDB<Post>('http://admin:admin@localhost:5984/post')
      if (remoteDB) {
        console.log("Connected to remote collection 'post'")
      } else {
        console.warn('Remote database connection failed')
      }
      this.storage = remoteDB

      // Base locale
      const localDB = new PouchDB<Post>('local_db')
      console.log("Initialized local database 'local_db'")

      // Synchronisation live entre la base distante et locale
      PouchDB.sync(remoteDB, localDB, { live: true, retry: true })
        .on('change', (info) => {
          console.log('Sync change detected:', info)
          this.getPosts() // Actualise les données locales
        })
        .on('error', (err) => {
          console.error('Sync error:', err)
        })

      // On utilise la base locale comme source principale
      this.storage = localDB

      // Créer un index au démarrage pour le champ post_name
      this.createIndex()
    },

    // Méthode pour synchroniser manuellement la base locale et distante
    manualSync() {
      if (!this.storage) {
        console.warn('Database is not initialized')
        return
      }

      const remoteDB = new PouchDB('http://admin:admin@localhost:5984/post')
      const localDB = new PouchDB('local_db')

      PouchDB.sync(remoteDB, localDB)
        .on('complete', () => {
          console.log('Manual synchronization completed')
        })
        .on('error', (err) => {
          console.error('Manual synchronization failed:', err)
        })
    },
    //méthode getPosts pour récupérer les posts
    async getPosts() {
      const response = await this.storage?.allDocs<Post>({ include_docs: true, attachments: true })

      if (response) {
        this.posts = {
          rows: response.rows.map((row) => {
            if (row.doc) {
              if (!row.doc.media) {
                row.doc.media = []
              }
              return {
                id: row.id,
                key: row.key,
                value: { rev: row.value.rev },
                doc: row.doc as Post
              }
            }

            return {
              id: row.id,
              key: row.key,
              value: { rev: row.value.rev },
              doc: {
                _id: row.id,
                post_name: '',
                post_content: '',
                attributes: { creation_date: '' },
                media: []
              }
            }
          }),
          total_rows: response.total_rows,
          offset: response.offset ?? 0
        }
      }
    },
    // méthode createPost pour créer un post
    async createPost(e: Event) {
      e.preventDefault()
      e.stopPropagation()

      const id = Math.random().toString(36).substring(2)

      const post: Post = {
        _id: id,
        post_name: this.name,
        post_content: this.content,
        attributes: {
          creation_date: new Date().toISOString()
        },
        media: []
      }

      const doc = await this.storage?.put(post)

      if (doc && this.mediaFile) {
        await this.storage?.putAttachment(
          id,
          this.mediaFile.name,
          doc.rev,
          this.mediaFile,
          this.mediaFile.type
        )
      }

      this.getPosts()

      this.name = ''
      this.content = ''
    },
    //méthode generateFakeData pour générer des données factices
    async generateFakeData() {
      const fakePosts = Array.from({ length: 10 }).map(() => ({
        _id: Math.random().toString(36).substring(2),
        post_name: `Post_${Math.random().toString(36).substring(7)}`,
        post_content: `This is a fake post content for ${new Date().toISOString()}`,
        attributes: {
          creation_date: new Date().toISOString()
        },
        media: []
      }))

      try {
        for (const post of fakePosts) {
          await this.storage?.put(post)
        }
        console.log('Fake data successfully added!')
        this.getPosts() // Rafraîchir les posts affichés
      } catch (error) {
        console.error('Error while adding fake data:', error)
      }
    },
    // méthodes pour gérer les opérations CRUD
    async uploadMedia(file: File): Promise<string> {
      return new Promise((resolve) => {
        setTimeout(() => resolve(URL.createObjectURL(file)), 1000)
      })
    },
    //méthode deletePost pour supprimer un post
    async deletePost(postId: string, rev: string) {
      await this.storage?.remove(postId, rev)
      this.getPosts()
    },
    //méthode editPost pour modifier un post
    async editPost(postId: string, updatedContent: string) {
      const post = await this.storage?.get<Post>(postId)
      if (post) {
        post.post_content = updatedContent
        await this.storage?.put(post)
        this.getPosts()
      }
    },
    //méthode createIndex pour créer un index
    async createIndex() {
      await this.storage?.createIndex({
        index: {
          fields: ['post_name']
        }
      })
      console.log('Index created')
    },
    //méthode removeMediaFromPost pour supprimer un média d'un post
    async removeMediaFromPost(postId: string, mediaIndex: number) {
      const post = await this.storage?.get<Post>(postId)

      if (post) {
        if (Array.isArray(post.media) && post.media.length > 0) {
          post.media.splice(mediaIndex, 1)

          await this.storage?.put(post)
          this.getPosts()
        }
      }
    },
    //méthode searchPosts pour rechercher des posts
    async searchPosts() {
      const response = await this.storage?.find({
        selector: { post_name: { $regex: `^.*${this.searchQuery}.*` } }
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
    //méthode syncDatabase pour synchroniser la base de données
    syncDatabase() {
      const localDB = new PouchDB('local_db')

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

  // Méthode montée pour initialiser la base de données et récupérer les posts
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

    <label for="media">Ajouter un média</label>
    <input type="file" id="media" @change="handleFileChange" />

    <button>Envoyer</button>
  </form>

  <button @click="generateFakeData">Générer des données "fake"</button>
  <button @click="manualSync">Synchroniser les bases</button>

  <h2>Recherche</h2>
  <input type="text" v-model="searchQuery" @input="searchPosts" placeholder="Rechercher par nom" />

  <h1>Posts</h1>

  <div v-if="posts && posts.rows.length">
    <div v-for="post in posts.rows" :key="post.id">
      <a v-if="post.doc?.post_name" :href="'/posts/' + post.doc._id">
        {{ post.doc.post_name }}
      </a>
      <p v-if="post.doc?.post_content">{{ post.doc.post_content }}</p>

      <div v-if="post.doc && post.doc.media && post.doc.media.length > 0">
        <h3>Médias associés:</h3>
        <div v-for="(media, index) in post.doc.media" :key="index">
          <img :src="media" alt="Media" width="100" />
          <button @click="removeMediaFromPost(post.id, index)">Supprimer le média</button>
        </div>
      </div>

      <div class="buttons">
        <button @click="deletePost(post.id, post.value.rev)">Supprimer</button>
        <button @click="editPost(post.id, promptEdit(post.doc.post_content))">Modifier</button>
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
