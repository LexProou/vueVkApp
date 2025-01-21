<template>
  <div class="min-h-screen bg-gray-100 p-4">
    <div v-if="!isLoggedIn">
      <button @click="vkLogin"
        class="bg-blue-500 hover:bg-blue-600 text-white font-medium py-2 px-4 rounded transition duration-300">
        Войти через VK
      </button>
    </div>

    <div v-if="isLoggedIn">
      <!-- Поиск пользователей -->
      <div class="my-4">
        <input v-model="searchQuery" @input="searchUsers"
          placeholder="Поиск по имени, фамилии, ID или имени пользователя"
          class="w-full p-2 border border-gray-300 rounded focus:ring-2 focus:ring-blue-500 focus:border-transparent" />
      </div>

      <!-- Результат поиска -->
      <div v-if="searchResults.length > 0" class="space-y-2">
        <ul>
          <li v-for="user in searchResults" :key="user.id" class="bg-white rounded-lg shadow p-3 flex items-center">
            <img :src="user.photo_100 || user.photo_50" alt="user avatar"
              class="w-10 h-10 rounded-full object-cover border-2 border-gray-200 hover:border-blue-500 transition-colors" />
            <span class="flex-grow mx-3">{{ user.first_name }} {{ user.last_name }} (ID: {{ user.id }})</span>
            <button @click="addUser(user)" class="bg-green-500 hover:bg-green-600 text-white px-3 py-1 rounded">
              Добавить
            </button>
          </li>
        </ul>
      </div>

      <!-- "Исходный" список -->
      <h3 class="text-xl font-semibold my-5">Исходный список</h3>
      <ul class="space-y-2">
        <li v-for="user in sourceList" :key="user.id"
          class="bg-white rounded-lg shadow p-3 flex items-center justify-between">
          <span>{{ user.first_name }} {{ user.last_name }} (ID: {{ user.id }})</span>
          <button @click="removeUser(user)" class="bg-red-500 hover:bg-red-600 text-white px-3 py-1 rounded">
            Удалить
          </button>
        </li>
      </ul>

      <!-- Строим список друзей -->
      <button @click="buildFriendsList"
        class="w-full my-5 bg-blue-500 hover:bg-blue-600 disabled:bg-gray-400 disabled:cursor-not-allowed text-white py-2 px-4 rounded transition duration-300"
        :disabled="sourceList.length === 0">
        Построить список друзей
      </button>

      <!-- Список друзей -->
      <div v-if="friendsList.length > 0" class="mt-8">
        <h3 class="text-xl font-semibold mb-4">Список друзей</h3>
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
          <div v-for="friend in sortedFriendsList" :key="friend.id"
            class="p-4 rounded-lg shadow cursor-pointer transform hover:-translate-y-1 transition duration-200"
            :style="{ backgroundColor: getFriendColor(friend.frequency) }" @click="showFriendDetails(friend)">
            <img :src="friend.photo_100" alt="friend avatar"
              class="w-24 h-24 rounded-full mx-auto mb-3 object-cover border-4 border-white shadow-lg hover:border-blue-500 transition-colors" />
            <div class="space-y-1">
              <h4 class="font-medium">{{ friend.last_name }} {{ friend.first_name }}</h4>
              <p class="text-gray-700">Пол: {{ friend.sex === 1 ? 'Ж' : 'М' }}</p>
              <p class="text-gray-700">Возраст: {{ friend.age || 'Не указан' }}</p>
              <p class="text-gray-700">Друзей: {{ friend.friends_count || 'Не указано' }}</p>
              <p class="text-gray-700">Встречается в списках: {{ friend.frequency }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Модальное окно с информацией о друге -->
      <div v-if="showingDetails" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center"
        @click="closeModal">
        <div class="bg-white rounded-lg p-6 max-w-2xl w-full max-h-[80vh] overflow-y-auto m-4" @click.stop>
          <div class="flex items-center mb-6">
            <button class="text-blue-500 border border-blue-500 px-3 py-1 rounded hover:bg-blue-50 mr-4"
              @click="closeModal">
              ← Назад
            </button>
            <h2 class="text-xl font-semibold">{{ selectedFriend.first_name }} {{ selectedFriend.last_name }}</h2>
          </div>

          <h3 class="font-medium mb-2">Находится в списках у:</h3>
          <ul class="mb-4">
            <li v-for="source in friendSources" :key="source.id" class="py-1">
              {{ source.first_name }} {{ source.last_name }}
            </li>
          </ul>

          <h3 class="font-medium mb-2">Последние записи со стены:</h3>
          <div v-for="post in friendPosts" :key="post.id" class="border-b border-gray-200 py-3 last:border-0">
            <p class="mb-2">{{ post.text }}</p>
            <small class="text-gray-500">{{ new Date(post.date * 1000).toLocaleDateString() }}</small>
          </div>

          <button @click="showingDetails = false"
            class="mt-4 bg-blue-500 hover:bg-blue-600 text-white py-2 px-4 rounded">
            Закрыть
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      isLoggedIn: false,
      accessToken: null,
      searchQuery: '',
      searchResults: [],
      sourceList: [],
      friendsList: [],
      selectedFriend: null,
      friendSources: [],
      friendPosts: [],
      showingDetails: false,
    };
  },
  computed: {
    sortedFriendsList() {
      return [...this.friendsList].sort((a, b) => {
        if (a.last_name === b.last_name) {
          return a.first_name.localeCompare(b.first_name);
        }
        return a.last_name.localeCompare(b.last_name);
      });
    },
  },
  methods: {
    delay(ms) {
      return new Promise(resolve => setTimeout(resolve, ms));
    },

    vkLogin() {
      VK.init({
        apiId: import.meta.env.VITE_VK_ID,
      });

      VK.Auth.login((response) => {
        if (response.session) {
          this.isLoggedIn = true;
          this.accessToken = response.session.access_token;
          console.log('Авторизация прошла успешно');
        } else {
          console.log('Ошибка авторизации');
        }
      }, VK.access.FRIENDS | VK.access.PHOTOS);
    },

    searchUsers() {
      if (this.searchQuery.trim().length < 3) {
        this.searchResults = [];
        return;
      }

      VK.Api.call('users.search', {
        q: this.searchQuery,
        access_token: this.accessToken,
        v: '5.211',
        count: 10,
        fields: 'photo_50,photo_100',
      }, (response) => {
        if (response.response) {
          const newResults = response.response.items.filter(user =>
            !this.sourceList.some(u => u.id === user.id)
          );
          this.searchResults = newResults;
        } else {
          console.log('Ошибка поиска:', response);
        }
      });
    },

    addUser(user) {
      this.sourceList.push(user);
      this.searchQuery = '';
      this.searchResults = [];
    },

    removeUser(user) {
      this.sourceList = this.sourceList.filter(u => u.id !== user.id);
      this.friendsList = [];
    },

    async buildFriendsList() {
      const friendsMap = new Map();

      for (const sourceUser of this.sourceList) {
        try {
          const response = await this.getFriendsData(sourceUser.id);
          if (response && response.items) {
            response.items.forEach(friend => {
              if (friendsMap.has(friend.id)) {
                const existing = friendsMap.get(friend.id);
                existing.frequency += 1;
                friendsMap.set(friend.id, existing);
              } else {
                friend.frequency = 1;
                friendsMap.set(friend.id, friend);
              }
            });
          }
        } catch (error) {
          console.error('Error fetching friends:', error);
        }
      }

      this.friendsList = Array.from(friendsMap.values());
      await this.enrichFriendsData();
    },

    getFriendsData(userId) {
      return new Promise((resolve, reject) => {
        VK.Api.call('friends.get', {
          user_id: userId,
          fields: 'photo_100,sex,bdate',
          v: '5.211'
        }, (response) => {
          if (response.response) {
            resolve(response.response);
          } else {
            reject(response.error);
          }
        });
      });
    },

    async enrichFriendsData() {
      for (let friend of this.friendsList) {
        if (friend.bdate) {
          const [day, month, year] = friend.bdate.split('.');
          if (year) {
            const birthDate = new Date(year, month - 1, day);
            const age = Math.floor((new Date() - birthDate) / (365.25 * 24 * 60 * 60 * 1000));
            friend.age = age;
          }
        }

        // Получаем количество друзей для каждого друга
        let retryCount = 0;
        const maxRetries = 5;
        while (retryCount < maxRetries) {
          try {
            const friendsCountResponse = await this.getFriendsCount(friend.id);
            friend.friends_count = friendsCountResponse.count;
            break;
          } catch (error) {
            if (error.error_code === 30) {
              console.warn(`Профиль пользователя ${friend.id} закрыт.`);
              friend.friends_count = 'Профиль приватный';
              break;
            } else if (error.error_code === 18) {
              console.warn(`Пользователь ${friend.id} был удален или заблокирован.`);
              friend.friends_count = 'Пользователь заблокирован или удален';
              break;
            } else if (error.error_code === 6) {
              console.warn('Слишком много запросов. Попробую снова...');
              retryCount++;
              await this.delay(1000);
            } else {
              console.error('Ошибка при получении количества друзей:', error);
              friend.friends_count = 'Не указано';
              break;
            }
          }
        }

        if (retryCount === maxRetries) {
          console.error(`Не удалось получить количество друзей для пользователя ${friend.id} после нескольких попыток.`);
        }
      }
    },

    getFriendsCount(userId) {
      return new Promise((resolve, reject) => {
        VK.Api.call('friends.get', {
          user_id: userId,
          v: '5.211'
        }, (response) => {
          if (response.response) {
            resolve({ count: response.response.count });
          } else {
            reject(response.error);
          }
        });
      });
    },

    showFriendDetails(friend) {
      this.selectedFriend = friend;
      VK.Api.call('friends.getLists', {
        user_id: friend.id,
        v: '5.211'
      }, (response) => {
        if (response.response) {
          this.friendSources = response.response;
        }
      });

      VK.Api.call('wall.get', {
        owner_id: friend.id,
        count: 5,
        v: '5.211'
      }, (response) => {
        if (response.response) {
          this.friendPosts = response.response.items;
        }
      });

      this.showingDetails = true;
    },

    closeModal() {
      this.showingDetails = false;
      this.friendSources = [];
      this.friendPosts = [];
    },

    getFriendColor(frequency) {
      if (frequency === 1) {
        return '#a7f3d0';
      } else if (frequency === 2) {
        return '#32aaba';
      } else {
        return '#15558a';
      }
    },
  },
};
</script>
