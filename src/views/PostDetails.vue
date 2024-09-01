<template>
  <div v-if="post" class="post-details">
    <h1 class="post-title">{{ post.title }}</h1>
    
    <p class="post-content">{{ post.content }}</p>

    <div v-if="post.image" class="post-image">
      <img :src="post.image" alt="Post Image" />
    </div>
    
    <p class="post-author" v-if="post.user">Written by: {{ post.user.name }}</p>

    <!-- Comment Form -->
    <div class="comment-form">
      <textarea v-model="newComment" placeholder="Add a comment..." rows="4" class="comment-textarea"></textarea>
      <button @click="submitComment" class="button submit-button">Submit</button>
    </div>

    <!-- Comments Section -->
    <div class="comments-section" v-if="comments.length">
      <h2>Comments</h2>
      <div v-for="comment in sortedComments" :key="comment.id" class="comment">
        <p class="comment-content">{{ comment.content }}</p>
        <p class="comment-author">By: {{ comment.user.name }}</p>
        <comment-actions 
            :isPostOwner="isCommentOwner(comment)"
            :commentId="comment.id"
            :commentSlug="comment.slug"
           @edit="editComment"
          @delete="deleteComment"
        />
      </div>
    </div>

    <!-- Editing Comment -->
    <div v-if="editingComment">
      <textarea v-model="editedCommentContent" rows="4" class="comment-textarea"></textarea>
      <button @click="updateComment" class="button submit-button">Save</button>
      <button @click="cancelEdit" class="button cancel-button">Cancel</button>
    </div>
  </div>
  <div v-else>
    <p>Loading post details...</p>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import axios from 'axios';
import type { Post, Comment, User } from '@/types';
import CommentActions from '@/components/CommentActions.vue'; 

export default defineComponent({
  
  props: {
    isPostOwner: {
      type: Boolean,
      required: true
    },
    commentId: {
      type: Number,
      required: true
    }
  },
  components: {
    CommentActions
  },
  data() {
    return {
      post: null as Post | null,
      newComment: '',
      comments: [] as Comment[],
      editingComment: null as Comment | null,
      editedCommentContent: '',
      user: null as User | null,
    };
  },
  computed: {
    isPostOwner() {
      return this.user && this.user.id === this.post?.user?.id;
    },
    isCommentOwner() {
      return (comment: Comment) => {
        return this.user && this.user.id === comment.user.id;
      };
    },
    sortedComments() {
      return this.comments.slice().sort((a, b) => {
        return new Date(b.created_at).getTime() - new Date(a.created_at).getTime();
      });
    }
  },
  methods: {
    onEdit() {
    console.log('Edit clicked for commentId:', this.commentId);
    this.$emit('edit', this.commentId);
  },
  onDelete() {
    console.log('Delete clicked for commentId:', this.commentId);
    this.$emit('delete', this.commentId);
  },
    async fetchUser() {
      const authToken = localStorage.getItem('authToken');
      if (!authToken) {
        alert('No authentication token found. Please log in.');
        return;
      }

      try {
        const response = await axios.get('https://interns-blog.nafistech.com/api/user', {
          headers: {
            Authorization: `Bearer ${authToken}`,
          }
        });
        this.user = response.data.data;
      } catch (error) {
        console.error('Error fetching user:', error.response ? error.response.data : error.message);
        if (error.response && error.response.status === 401) {
          alert('Unauthorized. Please log in again.');
          // Optionally, redirect to login page
        }
      }
    },

    async fetchPostDetails() {
      const slug = this.$route.params.slug;
      try {
        const response = await axios.get(`https://interns-blog.nafistech.com/api/posts/${slug}`, {
          headers: {
            Authorization: `Bearer ${localStorage.getItem('authToken')}`,
          }
        });
        this.post = response.data.data as Post;
        this.comments = response.data.data.comments || [];
       
        console.log('Fetched post details:', this.post);
        console.log('Fetched comments:', this.comments); 
      } catch (error) {
        console.error('Error fetching post details:', error.response ? error.response.data : error.message);
      }
    },

    async submitComment() {
  const postSlug = this.post?.slug;
  if (!postSlug || !this.newComment.trim()) {
    console.error('Post slug is missing or comment content is empty');
    return;
  }

  try {
    const response = await axios.post(
      `https://interns-blog.nafistech.com/api/posts/${postSlug}/comments`,
      {
        content: this.newComment.trim(),
      },
      {
        headers: {
          Authorization: `Bearer ${localStorage.getItem('authToken')}`,
          'Content-Type': 'application/json',
        },
      }
    );

   
    this.comments.push(response.data.data);

 
    this.newComment = '';
  } catch (error) {
    console.error('Error submitting comment:', error.response ? error.response.data : error.message);
  }
},



  async deleteComment(commentId: number) {
  const postSlug = this.post?.slug;
  if (!postSlug || !commentId) {
    console.error('Invalid postSlug or comment ID');
    return;
  }

  try {
    const response = await axios.delete(`https://interns-blog.nafistech.com/api/posts/${postSlug}/comments/${commentId}`, {
      headers: {
        Authorization: `Bearer ${localStorage.getItem('authToken')}`,
      },
    });
    
    if (response.status === 200) {
      this.comments = this.comments.filter(comment => comment.id !== commentId);
    } else {
      console.error('Unexpected response status:', response.status);
    }
  } catch (error) {
    console.error('Error deleting comment:', error.response ? error.response.data : error.message);
  }
},


    editComment(comment: Comment) {
      this.editingComment = comment;
      this.editedCommentContent = comment.content;
    },

    async updateComment() {
      if (!this.editingComment) return;

      try {
        const response = await axios.put(
          `https://interns-blog.nafistech.com/api/comments/${this.editingComment.id}`,
          { content: this.editedCommentContent },
          {
            headers: {
              Authorization: `Bearer ${localStorage.getItem('authToken')}`,
              'Content-Type': 'application/json',
            },
          }
        );

        const updatedComment = response.data;
        const index = this.comments.findIndex(c => c.id === updatedComment.id);
        if (index !== -1) {
          this.comments.splice(index, 1, updatedComment);
        }

        this.cancelEdit(); // Reset the edit mode
      } catch (error) {
        console.error('Error updating comment:', error.response ? error.response.data : error.message);
      }
    },

    cancelEdit() {
      this.editingComment = null;
      this.editedCommentContent = '';
    },
  },
  mounted() {
    this.fetchUser();
    this.fetchPostDetails();
  }
});
</script>



<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@500;700&family=Roboto:wght@400;500&display=swap');

.post-details {
  min-height: 100vh; 
  background-image: url('@/assets/background.jpg'); 
  background-size: cover; 
  background-position: center;
  background-repeat: no-repeat; 
  color: #fff; 
  padding: 20px; 

  /* Apply the imported fonts */
  font-family: 'Roboto', sans-serif;
}

h1, h2, h3, .title {
  font-family: 'Montserrat', sans-serif; /* Use Montserrat for titles/headings */
  font-weight: 700; 
}

.post-title {
  font-family: 'Montserrat', sans-serif;
  font-size: 2.5em;
  color: #333333;
  margin-bottom: 20px;
  font-weight: bold;
  text-align: center;
  animation: fadeInDown 1s ease-out;
}

.post-content {
  font-family: 'Roboto', sans-serif;
  font-size: 1.8em;
  color: #1d1a1a;
  line-height: 1.6;
  margin-bottom: 20px;
  text-align: justify;
  animation: fadeIn 1.2s ease-out;
}

.post-image img {
  width: 100%;
  height: 500;
  object-fit: scale-down;
  border-radius: 8px;
  margin-bottom: 15px;
}

.post-author {
  font-family: 'Roboto', sans-serif;
  font-size: 1em;
  color: #3498db;
  margin-top: 20px;
  padding: 5px 10px;
  border-left: 4px solid #3498db;
  background-color: #f3f4f7;
  border-radius: 5px;
  animation: fadeIn 1s ease-out;
}

.comments-section {
  margin-top: 30px;
  animation: fadeInUp 1s ease-out;
}

.comment {
  font-family: 'Roboto', sans-serif;
  font-size: 1em;
  color: #333333;
  margin-bottom: 15px;
  padding: 10px 15px;
  background-color: #f9f9f9;
  border-radius: 5px;
  animation: fadeIn 1s ease-out;
}

.comment-content {
  font-family: 'Roboto', sans-serif;
  font-size: 1.2em;
  color: #333333;
  margin-bottom: 10px;
  animation: fadeInUp 1s ease-out;
}

.comment-author {
  font-family: 'Roboto', sans-serif;
  font-size: 1em;
  color: #3498db;
  margin-bottom: 10px;
}

.comment-textarea {
  width: 100%;
  font-family: 'Roboto', sans-serif;
  font-size: 1.1em;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 5px;
  border: 1px solid #dddddd;
  resize: vertical;
  box-sizing: border-box;
}

.button {
  font-family: 'Roboto', sans-serif;
  font-size: 1em;
  padding: 10px 20px;
  background-color: #3498db;
  color: #ffffff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.button:hover {
  background-color: #2980b9;
}

.submit-button {
  font-family: 'Montserrat', sans-serif;
  font-weight: bold;
}

.cancel-button {
  background-color: #e74c3c;
  margin-left: 10px;
}

.cancel-button:hover {
  background-color: #c0392b;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
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