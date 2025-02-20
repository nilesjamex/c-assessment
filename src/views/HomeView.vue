<template>
  <section>
    <div class="home">
      <!-- Search Status Indicator -->
      <div v-if="shouldShowStatus" class="home__status">
        <p class="home__status__text" :class="{ searching: loading }">
          {{ loading ? "Searching for" : "Search results for" }}
          <span class="keyword">"{{ activeSearchTerm }}"</span>
        </p>
      </div>

      <div class="home__search">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search for photo"
          class="search-input"
        />
        <div
          v-if="isDebouncing && searchQuery.length > 0"
          class="home__spinner__wrapper"
        >
          <svg class="home__spinner" width="16" height="16" viewBox="0 0 24 24">
            <circle
              class="home__spinner__circle"
              cx="12"
              cy="12"
              r="10"
              fill="none"
              stroke="currentColor"
              stroke-width="3"
            />
          </svg>
        </div>
        <div class="search-icon">
          <svg
            class="icon"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              strokeLinecap="round"
              strokeLinejoin="round"
              strokeWidth="2"
              d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
            />
          </svg>
        </div>
      </div>

      <!-- Shimmer Loading State -->
      <div v-if="loading" class="home__grid">
        <div v-for="n in 8" :key="n" class="home__cards home__shimmer">
          <div class="home__shimmer__img"></div>
          <div class="home__shimmer__content">
            <div class="home__shimmer__line"></div>
            <div class="home__shimmer__line home__shimmer__line__short"></div>
          </div>
        </div>
      </div>

      <!-- empty array -->
      <div v-else-if="images.length === 0" class="no-results">
        No images found. Try a different search term.
      </div>

      <!-- Profile Grid -->
      <div class="home__grid" v-else>
        <div
          v-for="(image, index) in images"
          :key="image.id"
          @click="openModal(image)"
          :class="['home__cards', getCardClass(index)]"
        >
          <!-- :style="getCardStyle(index)" -->
          <div class="home__cards__image">
            <img :src="image.urls.regular" :alt="photo" />
          </div>
          <div class="home__cards__info">
            <h3 class="home__cards__name">
              {{ image?.user?.first_name }} {{ image?.user?.last_name }}
            </h3>
            <p class="home__cards__location">{{ image?.user?.location }}</p>
          </div>
        </div>
      </div>
    </div>
    <!-- Modal -->
    <Teleport to="body">
      <div v-if="selectedImage" class="modal-overlay" @click="closeModal">
        <div class="modal" @click.stop>
          <button class="close-button" @click="closeModal">
            <svg
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
            >
              <path d="M18 6L6 18M6 6l12 12"></path>
            </svg>
          </button>

          <div class="modal-image">
            <img
              :src="selectedImage.urls.regular"
              :alt="selectedImage.alt_description"
            />
          </div>

          <div class="modal-content">
            <h2 class="modal-title">{{ selectedImage.user.name }}</h2>
            <p class="modal-location">
              {{ selectedImage.location?.city || "Unknown location" }},
              {{ selectedImage.location?.country || "" }}
            </p>

            <div class="modal-details">
              <p class="modal-description">
                {{
                  selectedImage.description ||
                  selectedImage.alt_description ||
                  "No description available"
                }}
              </p>

              <div class="modal-stats">
                <div class="stat">
                  <span class="stat-label">Likes</span>
                  <span class="stat-value">{{ selectedImage.likes }}</span>
                </div>
                <div class="stat">
                  <span class="stat-label">Downloads</span>
                  <span class="stat-value">{{
                    selectedImage.downloads || "N/A"
                  }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Teleport>
  </section>
</template>

<script setup>
import { ref, onMounted, watch, onUnmounted } from "vue";
import axios from "axios";

const accessKey = process.env.VUE_APP_UNSPLASH_KEY;
const images = ref([]);
const loading = ref(true);
const error = ref(null);
const searchQuery = ref("");
const searchTerm = ref("Africa");
const activeSearchTerm = ref("Africa");
let debounceTimeout = null;
const isDebouncing = ref(false);
const shouldShowStatus = ref(false);
let statusTimeout = null;

const fetchImages = async (query) => {
  try {
    loading.value = true;
    error.value = null;
    activeSearchTerm.value = query;
    if (searchQuery.value.length > 0) {
      shouldShowStatus.value = true;
    }

    try {
      const response = await axios.get(
        "https://api.unsplash.com/search/photos",
        {
          params: {
            query: searchTerm.value,
            per_page: 9,
            content_filter: "high",
          },
          headers: {
            Authorization: `Client-ID ${accessKey}`,
          },
        }
      );
      images.value = response.data.results;
    } catch (error) {
      throw new Error("Failed to fetch images from Unsplash");
    }
  } catch (err) {
    error.value = err.message;
    console.error("Error fetching images:", err);
  } finally {
    loading.value = false;
    isDebouncing.value = false;
  }
};

// search keyword
watch(searchQuery, (newValue) => {
  searchTerm.value = newValue;
  if (newValue.trim()) {
    statusTimeout = setTimeout(() => {
      shouldShowStatus.value = true;
    }, 1800);
  }
});

watch(searchTerm, (newValue) => {
  if (debounceTimeout) {
    clearTimeout(debounceTimeout);
  }
  if (statusTimeout) {
    clearTimeout(statusTimeout);
  }

  if (newValue.trim()) {
    isDebouncing.value = true;

    debounceTimeout = setTimeout(() => {
      fetchImages(newValue);
    }, 2000);
  } else {
    shouldShowStatus.value = false;
  }
});

const getCardClass = (index) => {
  // Map the index to the correct class
  if (index % 3 === 0) return "left";
  if (index % 3 === 1) return "middle";
  if (index % 3 === 2) return "right";
};

const getCardStyle = (index) => {
  const row = Math.floor(index / 3) + 1;
  const column = (index % 3) + 1;
  return {
    gridRow: row,
    gridColumn: column,
  };
};

onMounted(() => {
  fetchImages(searchQuery.value);
});

// modal state
const selectedImage = ref(null);

const openModal = (image) => {
  selectedImage.value = image;
  document.body.style.overflow = "hidden";
};

const closeModal = () => {
  selectedImage.value = null;
  document.body.style.overflow = "";
};

// Close modal on escape key
const handleKeydown = (e) => {
  if (e.key === "Escape" && selectedImage.value) {
    closeModal();
  }
};

onMounted(() => {
  document.addEventListener("keydown", handleKeydown);
});

onUnmounted(() => {
  document.removeEventListener("keydown", handleKeydown);
});
</script>

<style lang="scss" scoped>
@keyframes shimmer {
  0% {
    background-position: -1000px 0;
  }
  100% {
    background-position: 1000px 0;
  }
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

@keyframes slideDown {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.home {
  margin: 0 auto;
  &__search {
    position: relative;
    margin-bottom: 2rem;
    height: 25vh;
    background-color: $secondary-color;
    padding: 2rem;
    display: grid;
    place-items: center;
    & > div {
      position: absolute;
      right: 4%;
      top: 50%;
      @include respondMax("tablet") {
        right: 3rem;
      }
      &.search-icon {
        position: absolute;
        left: 4%;
        top: 50%;
        @include respondMax("tablet") {
          left: 3rem;
        }
      }
    }
    svg {
      transform: translateY(-50%);
      color: $black;
      width: 1.5rem;
      height: 1.5rem;
    }
    input {
      width: 100%;
      height: Max(64px, 4rem);
      padding: 1rem 1rem 1rem 5rem;
      border-radius: $border-radius;
      border: 1px solid $grey;
      background-color: $white;
      font-size: Max(1.125rem, 18px);
      transition: box-shadow $transition-duration ease;
      border-radius: 8px;
      color: $black;
      font-weight: normal;
      font-family: Hk-Grotesk, "sans-serif";

      &:focus {
        outline: none;
        box-shadow: 0 0 0 2px rgba($primary-color, 0.1);
      }
    }
  }
  &__grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-auto-rows: 300px;
    grid-auto-flow: dense;
    grid-gap: 10px;
    gap: 5rem;
    padding: 0 2rem;
    margin-top: -4rem;
    @include respondMax("tablet") {
      grid-template-columns: repeat(2, 1fr);
    }
  }
  &__cards {
    position: relative;
    border-radius: $border-radius;
    overflow: hidden;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    @include hover-lift;
    &.left {
      height: 350px;
      // grid-row: 3;
      // &:not(:first-child) {
      //   margin-top: -10rem;
      // }
    }

    &.middle {
      height: 450px;
      // grid-row: 4;
    }

    &.right {
      height: 400px;
      // grid-row: 3;
    }
    &__image {
      position: relative;
      padding-top: 125%;
      & > img {
        position: absolute;
        top: 0;
        left: 0;
        width: Min(100%, 100%);
        height: 100%;
        object-fit: cover;
      }
    }
    &__info {
      position: absolute;
      bottom: 0;
      left: 0;
      right: 0;
      padding: 1.5rem;
      background: linear-gradient(to top, $overlay-color, transparent);
      color: white;
    }
    &__name {
      font-size: 1.25rem;
      font-weight: 600;
      margin-bottom: 0.25rem;
    }
    &__location {
      font-size: 0.875rem;
      opacity: 0.9;
    }
  }
  &__shimmer {
    background: white;
    &__img {
      width: 100%;
      height: 250px;
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 1000px 100%;
      animation: shimmer 2s infinite linear;
    }
    &__content {
      padding: 12px;
    }
    &__line {
      height: 15px;
      margin-bottom: 10px;
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 1000px 100%;
      animation: shimmer 2s infinite linear;
      border-radius: 4px;
      &__short {
        width: 60%;
      }
    }
  }
  &__spinner {
    animation: rotate 1s linear infinite;
    &__wrapper {
      position: absolute;
      right: 12px;
      top: 50%;
      transform: translateY(-50%);
      color: #666;
    }
    &__circle {
      stroke-linecap: round;
      stroke-dasharray: 60;
      stroke-dashoffset: 20;
    }
  }
  &__status {
    background-color: $secondary-color;
    padding: 1rem 2rem;
    width: 100%;
    text-align: center;
    opacity: 0;
    transform: translateY(-10px);
    animation: slideDown 0.3s forwards;
    &__text {
      font-size: Max(3.5rem, 32px);
      color: #666;
      margin-top: 1rem;
      text-align: center;
      &.searching {
        color: #2196f3;
      }
      .keyword {
        font-weight: 600;
        color: #333;
      }
    }
  }
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.75);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 20px;
}

.modal {
  background: white;
  border-radius: 12px;
  max-width: 800px;
  width: 100%;
  position: relative;
  overflow: hidden;
  animation: modalFade 0.3s ease-out;
}

@keyframes modalFade {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.close-button {
  position: absolute;
  top: 16px;
  right: 16px;
  background: white;
  border: none;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  z-index: 1;
}

.modal-image {
  width: 100%;
  height: 400px;
  position: relative;
}

.modal-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.modal-content {
  padding: 24px;
}

.modal-title {
  font-size: 24px;
  font-weight: 600;
  margin: 0;
}

.modal-location {
  color: #666;
  margin: 8px 0 20px;
  font-size: 14px;
}

.modal-details {
  display: grid;
  gap: 20px;
}

.modal-description {
  margin: 0;
  line-height: 1.6;
}

.modal-stats {
  display: flex;
  gap: 24px;
}

.stat {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.stat-label {
  font-size: 12px;
  color: #666;
}

.stat-value {
  font-weight: 600;
}
</style>
