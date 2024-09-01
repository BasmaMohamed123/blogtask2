<template>
    <div class="create-post-container">
      <h2 class="header">Create a New Post</h2>
      <form @submit.prevent="submitPost" >
        <div>
          <label for="title">Title</label>
          <input
            id="title"
            name="title" 
            v-model="post.title"
            type="text"
            placeholder="Enter post title"
            required
          />
        </div>
      <div>
        <label for="content">Content</label>
        <textarea
          id="content"
           name="content"
          v-model="post.content"
          placeholder="Enter post content"
          required
        ></textarea>
      </div>

        <div>
          <label for="file">Upload File:</label>
          <input type="file" @change="handleFileChange" id="file" />
          <div v-if="imagePreview" class="imagePreview">
            <img :src="imagePreview" alt="Preview" />
            
          </div>
        </div>

        <button type="submit" class="submit-button">Submit Post</button>
      </form>
      <nav class="navigation">
        <router-link to="/" class="nav-link">Home</router-link>
        <router-link to="/blog-posts" class="nav-link">View Blog Posts</router-link>
      
      </nav>
    </div>
  </template>



  <script lang="ts">
  import { defineComponent ,ref } from 'vue';
  import axios from 'axios';
  import { Post, ApiResponse } from './types';

  
  export default defineComponent({
    name: 'CreatePost',
    setup() {
      const post = ref<Post>({ title: '', content: '', image: '' });
    const image = ref<File | null>(null);
      const imagePreview = ref<string | null>(null);




      const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    image.value = target.files[0];
    // Generate a preview of the image
    const reader = new FileReader();
    reader.onload = (e) => {
      imagePreview.value = e.target?.result as string;
    };
    reader.readAsDataURL(target.files[0]);
  }
};

    const submitPost = async () => {
  const formData = new FormData();
  formData.append('title', post.value.title);
  formData.append('content', post.value.content);
  
  if (image.value) {
    formData.append('file', image.value); 
  }

  

  try {
    const response = await axios.post<ApiResponse<{ image: string }>>('https://interns-blog.nafistech.com/api/posts/', formData, {
      headers: {
        'Content-Type': 'multipart/form-data',
        Authorization: `Bearer ${localStorage.getItem('authToken')}`,
      },
    });
    post.value.image = response.data.data.image;
    console.log('Post created successfully:', response.data.message);
    post.value = { title: '', content: '', image: '' };
    image.value = null;
    imagePreview.value = null;
  } catch (error) {
    console.error('Error creating post:', error.response ? error.response.data : error.message);
  }
};

   
    return {
       title: '',
          content: '',
      post,
      image,
      imagePreview,
      handleFileChange,
      submitPost,
    };
  },

    
    data() {
      return {
        newPost: {
          title: '',
          content: '',
          image: '',
        }
      };
    },
   
    async submitPost() {
      const authToken = localStorage.getItem('authToken');
      if (!authToken) {
        alert('You must be logged in to create a post.');
        return;
      }

      const url = 'https://interns-blog.nafistech.com/api/posts'; 

      try {
        const response = await axios.post(
          url,
          {
            title: this.newPost.title,
            content: this.newPost.content, //image
            image: this.newPost.image,
          },
          {
            headers: {
              Authorization: `Bearer ${authToken}`,
              'Content-Type': 'application/json',
            }
          }
        );
        this.$router.push('/posts'); 

        console.log('Post created:', response.data.data);
        this.newPost.title = '';
        this.newPost.content = '';
        
       
      
      } catch (error) {
        console.error('Error creating post:', error.response ? error.response.data.data : error.message);
        alert('Failed to create post. Please try again.');
      }
    
  },
  mounted() {
    console.log('CreatePostView is mounted');
  }

  });
  </script>
  

  <style scoped>
  @import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@500;700&family=Roboto:wght@400;500&display=swap');
  
  .create-post-container {
    min-height: 100vh; 
    background-image: url('@/assets/background.jpg'); 
    background-size: cover; 
    background-position: center;
    background-repeat: no-repeat; 
    color: #813d3d; 
    padding: 20px; 
  
  

  }
  
  .header {
    font-family: 'Montserrat', sans-serif;
    font-size: 2em;
    color: #333333;
    margin-bottom: 20px;
    text-align: center;
    animation: fadeInDown 1s ease-out;
  }
  
  label {
    display: block;
    font-family: 'Roboto', sans-serif;
    font-size: 1.1em;
    color: #333333;
    margin-bottom: 5px;
    font-weight: 500;
  }
  
  input[type="text"],
  textarea {
    width: 100%;
    font-family: 'Roboto', sans-serif;
    font-size: 1em;
    padding: 10px;
    margin-bottom: 15px;
    border-radius: 5px;
    border: 1px solid #800606;
    box-sizing: border-box;
    transition: border-color 0.3s;
  }
  
  input[type="text"]:focus,
  textarea:focus {
    border-color: #6e8a9c;
    outline: none;
  }
  
  .image-preview {
    margin-top: 15px;
    text-align: center;
  }
  
  .image-preview img {
    max-width: 100%;
    height: auto;
    border-radius: 8px;
  }
  
  .submit-button {
    font-family: 'Montserrat', sans-serif;
    font-size: 1.2em;
    font-weight: bold;
    padding: 10px 20px;
    background-color: #3498db;
    color: #ffffff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s;
    display: block;
    width: 100%;
    margin-top: 20px;
  }
  
  .submit-button:hover {
    background-color: #2980b9;
  }
  
  .navigation {
    margin-top: 30px;
    text-align: center;
  }
  
  .nav-link {
    font-family: 'Roboto', sans-serif;
    font-size: 1.1em;
    color: #3498db;
    text-decoration: none;
    margin: 0 10px;
    transition: color 0.3s;
  }
  
  .nav-link:hover {
    color: #2980b9;
  }
  
  @keyframes fadeInDown {
    from {
      opacity: 0;
      transform: translateY(-20px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
  </style>