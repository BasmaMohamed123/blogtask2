<template>
  <div class="blog-posts">
    <h1>My Blog Posts</h1>
    <div v-if="!isLoggedIn">
      <p>Please <router-link to="/login">login</router-link> to view your posts.</p>
    </div>
    <div v-else>
      <router-link to="/create-post" class="create-post-link">
        <button class="create-post-button">Create New Post</button>
      </router-link>

      <div class="search-container">
        <div class="search-box">
          <input
            v-model="searchQuery"
            type="text"
            class="search-input"
            placeholder="Search posts..."
          />
          <button @click="searchPosts" class="search-button">
            Search
          </button>
        </div>
      </div>

      <div class="sort-container">
        <button @click="sortPosts('latest')" :class="{ active: sortOrder === 'latest' }">Sort by Latest</button>
        <button @click="sortPosts('oldest')" :class="{ active: sortOrder === 'oldest' }">Sort by Oldest</button>
      </div>

      <div v-if="filteredPosts.length === 0">
        <p>No posts available.</p>
      </div>
      <div v-else>
        <div class="post-list">
          <div v-for="post in filteredPosts" :key="post.slug" class="post-item">
            <h2 class="post-title">{{ post.title }}</h2>
            <p class="post-excerpt">{{ post.excerpt }}</p>
            <p class="post-author" v-if="post.user && post.user.name">Author: {{ post.user.name }}</p>
            
            <!-- Display image if available -->
            <img v-if="post.image" :src="post.image" alt="Post Image" class="post-img" />
            
            <p v-else class="no-image">No image available</p>

            <p class="post-comments">Comments: {{ post.comments_count }}</p>
            <div class="post-actions">
              <router-link :to="`/posts/${post.slug}`">
                <button class="view-details">View Details</button>
              </router-link>

              <div v-if="isPostOwner(post)">
                <router-link :to="`/edit-post/${post.slug || post.id}`">
                  <button class="edit-post">Edit</button>
                </router-link>
                <button @click="confirmDelete(post.slug || post.id)" class="delete-post">Delete</button>
              </div>
            </div>

            <!-- Display the latest comment if available -->
            <div v-if="post && post.latestComment" class="last-comment">
              Latest Comment: {{ post.latestComment.content }}
            </div>
            <div v-else class="last-comment">
              No comments available.
            </div>
            
            
          </div>
        </div>

        <Pagination
          :currentPage="currentPage"
          :totalPages="totalPages"
          @page-changed="handlePageChange"
        />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, computed } from 'vue';
import axios from 'axios';
import { Post, User } from '../types'; // Import your types
import Pagination from '../components/pagination.vue';

export default defineComponent({
  name: 'BlogPosts',
  components: {
    Pagination,
  },
  setup() {
    const posts = ref<Post[]>([]);
    const searchQuery = ref('');
    const isLoggedIn = ref(false);
    const user = ref<User | null>(null);
    const currentUserName = ref('');
    const currentPage = ref(1);
    const totalPages = ref(1);
    const sortOrder = ref<'latest' | 'oldest'>('latest');

    const fetchPosts = async (page = 1) => {
      
      currentPage.value = page;
      try {
        const authToken = localStorage.getItem('authToken');
        const response = await axios.get(`https://interns-blog.nafistech.com/api/posts?page=${page}&sort=${sortOrder.value}`, {
          headers: {
            Authorization: `Bearer ${authToken}`,
          },
        });
        posts.value = response.data.data;
        totalPages.value = response.data.meta.last_page;
       
        // Fetch comments for each post
        for (const post of posts.value) {
          const commentsResponse = await axios.get(`https://interns-blog.nafistech.com/api/posts/${post.id}/comments`, {
            headers: {
              Authorization: `Bearer ${authToken}`,
            },
          });
          const comments = commentsResponse.data.data;
          if (comments.length > 0) {
            post.latestComment = comments[comments.length - 1];
          }
        }
      } catch (error) {
        console.error('Error fetching posts:', error);
      }
    };

    const fetchUser = async () => {
      try {
        const authToken = localStorage.getItem('authToken');
        const response = await axios.get('https://interns-blog.nafistech.com/api/user', {
          headers: {
            Authorization: `Bearer ${authToken}`,
          },
        });
        user.value = response.data;
        currentUserName.value = user.value.name;
        fetchPosts();
      } catch (error) {
        console.error('Error fetching user data:', error);
      }
    };

    onMounted(() => {
      isLoggedIn.value = !!localStorage.getItem('authToken');
      if (isLoggedIn.value) {
        fetchUser();
      }
    });

    const filteredPosts = computed(() => {
      const query = searchQuery.value.toLowerCase();
      if (!query) return posts.value;

      const calculateRelevance = (post: Post): number => {
        const title = post.title ? post.title.toLowerCase() : '';
        const excerpt = post.excerpt ? post.excerpt.toLowerCase() : '';

        const titleScore = title.includes(query) ? 2 : 0;
        const excerptScore = excerpt.includes(query) ? 1 : 0;
        return titleScore + excerptScore;
      };

      return posts.value
        .map(post => ({
          ...post,
          relevance: calculateRelevance(post),
        }))
        .filter(post => post.relevance > 0) // Filter out posts with zero relevance
        .sort((a, b) => b.relevance - a.relevance || (a.title || '').localeCompare(b.title || ''));
    });

    const isPostOwner = (post: Post): boolean => {
      return user.value !== null && post.user?.name === user.value.name;
    };

    const sortPosts = (order: 'latest' | 'oldest') => {
      sortOrder.value = order;
      fetchPosts();
    };

    const handlePageChange = (page: number) => {
      fetchPosts(page);
    };

    const confirmDelete = (postId: string | number) => {
  if (confirm('Are you sure you want to delete this post?')) {
    deletePost(postId);
  }
};


    const deletePost = async (postId: string | number) => {
      try {
        const authToken = localStorage.getItem('authToken');
        await axios.delete(`https://interns-blog.nafistech.com/api/posts/${postId}`, {
          headers: {
            Authorization: `Bearer ${authToken}`,
          },
        });
        posts.value = posts.value.filter(post => post.id !== postId);
      } catch (error) {
        console.error('Error deleting post:', error);
      }
    };

    return {
      posts,
      searchQuery,
      isLoggedIn,
      user,
      currentUserName,
      currentPage,
      totalPages,
      sortOrder,
      filteredPosts,
      isPostOwner,
      sortPosts,
      handlePageChange,
      confirmDelete,
      deletePost,
    };
  },
});
</script>






<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@500;700&family=Roboto:wght@400;500&display=swap');

.blog-posts {
  padding: 20px;
  background: url('@/assets/background.jpg') no-repeat center center fixed;
  background-size: cover;
  position: relative;
  overflow: hidden;
  width:auto;
}

.blog-posts::before {
  content: '';
  position: center;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  z-index: 1;
  animation: backgroundAnimation 20s linear infinite;
}

@keyframes backgroundAnimation {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

.blog-posts > * {
  position: center;
  z-index: 2;
}

h1 {
  font-family: 'Montserrat', sans-serif;
  color: #ffffff;
  font-size: 2.5em;
  margin-bottom: 30px;
  text-align: center;
  letter-spacing: 1px;
  animation: fadeInDown 1s ease-out;
  
}

.create-post-link {
  display: block;
  margin-top: 20px;
  text-align: center;
}

.create-post-button {
  background-color: #8146d3; /* Blue background */
  color: #ffffff; /* White text */
  font-size: 1.5em;
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.3s ease;
}

.create-post-button:hover {
  background-color: #995d7d; /* Darker blue on hover */
  transform: scale(1.05); /* Slightly enlarge the button */
}

.create-post-button:focus {
  outline: none; /* Remove default focus outline */
  box-shadow: 0 0 0 2px rgba(0, 0, 0, 0.2); /* Add custom focus outline */
}

.login-link {
  color: #3498db; /* Blue text */
  text-decoration: none;
}

.login-link:hover {
  text-decoration: underline; /* Underline on hover */
}

.post-list {
  display: flex;
  flex-direction: column;
  gap: 25px;
}

.post-item {
  background-color: rgba(255, 255, 255, 0.9);
  border: 1px solid #e0e0e0;
  border-radius: 10px;
  padding: 20px;
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.05);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  animation: fadeInUp 0.5s ease-out;
}

.post-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
}

.post-title {
  font-family: 'Montserrat', sans-serif;
  font-size: 1.75em;
  color: #333333;
  margin-bottom: 15px;
  transition: color 0.3s ease;
}

.post-item:hover .post-title {
  color: #0056b3;
}

.post-excerpt {
  font-family: 'Roboto', sans-serif;
  color: #666666;
  margin-bottom: 15px;
  line-height: 1.6;
  animation: fadeIn 1.2s ease-out;
}

.post-author {
  font-family: 'Roboto', sans-serif;
  color: #888888;
  font-style: italic;
  margin-bottom: 20px;
  animation: fadeIn 1.2s ease-out;
}

.post-actions {
  display: flex;
  gap: 15px;
  animation: fadeInUp 1s ease-out;
}

button {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-family: 'Montserrat', sans-serif;
  font-weight: 500;
  transition: background-color 0.3s ease, transform 0.3s ease;
}

button.view-details {
  background-color: #0056b3;
  color: white;
}

button.view-details:hover {
  background-color: #003e7e;
  transform: scale(1.05);
}

@keyframes fadeInDown {
  0% {
    opacity: 0;
    transform: translateY(-20px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes fadeInUp {
  0% {
    opacity: 0;
    transform: translateY(20px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}
@keyframes fadeIn {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}
.search-container {
  display: flex;
  justify-content: center;
  margin: 20px;
  
 
}
.search-input {
  flex:  1;
  padding: 10px;
  border: none;
  outline: none;
  font-size: 1.3em;
  color: #2d1111;
}

.search-button {
  padding: 10px 20px;
  border: none;
  background-color: #d3accc;
  color: rgb(23, 20, 22);
  font-size: 1em;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.search-button:hover {
  background-color: #d764df;
}
.post-img {
  width: auto;
  height: 200px;
  object-fit: scale-down;
  border-radius: 8px;
  margin-bottom: 15px;
}
.last-comment {
  font-style: italic;
  color: #555;
  margin-top: 10px;
}
.post-comments {
  font-family: 'Montserrat', sans-serif;
  font-size: 19px;
  font-weight: bold;
  color: #882a2a;
  margin-top: 20px;
  margin-bottom: 20px;
  padding: 10px;
  background-color: #f4f4f4;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

</style>
