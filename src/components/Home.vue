<template>
  <div class="container">
    <div class="row">
      <div class="eight columns offset-by-two">
        <div class="row">
          <div class="six columns offset-by-three">
            <div class="auth">
              <label for="usernameInput">Auth</label>
              <input
                class="u-full-width"
                type="text"
                placeholder="Username"
                id="usernameInput"
                v-model="auth.userName"
              />
              <input
                class="u-full-width"
                type="password"
                placeholder="Password"
                id="passwordInput"
                v-model="auth.password"
              />
            </div>
            <label for="beerIdInput">Beer</label>
            <input
              class="u-full-width"
              type="text"
              placeholder="Beer ID"
              id="beerIdInput"
              v-model="beerId"
            />
          </div>
          <div class="six columns offset-by-three">
            <button
              class="button-primary u-full-width"
              :disabled="
                !(
                  beerId &&
                  auth.userName &&
                  auth.password &&
                  auth.userName.length > 0 &&
                  auth.password.length > 0 &&
                  beerId.length > 0
                )
              "
              @click="onSubmit"
            >
              <span>{{ loading ? "Loading..." : "Fetch" }}</span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <div class="row">
      <div class="eight columns offset-by-two output" v-if="beerInfo">
        <h3>{{ beerInfo.beer_name }}, {{ beerInfo.brewery.brewery_name }}</h3>
        <div
          v-for="flavor in beerInfo.flavor_profile.items"
          :key="flavor.tag_id"
          class="flavors"
        >
          <div class="flavor">
            {{ flavor.tag_name }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "Home",
  data() {
    return {
      loading: false,
      client_id: process.env.VUE_APP_CLIENT_ID, // Android app
      client_secret: process.env.VUE_APP_CLIENT_SECRET, // Anroid app
      device_udid: "9b5e46525304d62a", // Pixel 3A (emulated device)
      auth: {
        userName: null,
        password: null,
        token: null,
      },
      beerId: null,
      beerInfo: null,
    };
  },
  created() {
    this.checkValidToken();
  },
  methods: {
    isFormValid() {
      return (
        !this.beerId &&
        !this.auth.userName &&
        !this.auth.password &&
        this.auth.userName.length > 0 &&
        this.auth.password.length > 0 &&
        this.beerId.length > 0
      );
    },
    moreThanOneHourAgo(date) {
      const HOUR = 1000 * 60 * 60;
      const anHourAgo = Date.now() - HOUR;

      return date < anHourAgo;
    },
    authUser() {
      const formData = new FormData();
      formData.append("user_name", this.auth.userName);
      formData.append("user_password", this.auth.password);
      formData.append("device_udid", this.device_udid);

      return this.axios({
        method: "POST",
        url: `https://cors.israndom.win/https://api.untappd.com/v4/xauth?client_id=${this.client_id}&client_secret=${this.client_secret}`,
        data: formData,
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
          "X-Requested-With": "com.untappdllc.app",
        },
      })
        .then((response) => {
          const access_token = response.data.response.access_token;
          localStorage.setItem("access_token", access_token);
          localStorage.setItem("access_token_checked", Date.now());
          this.auth.token = access_token;
        })
        .catch((error) => console.error(error));
    },
    checkValidToken() {
      const access_token = localStorage.getItem("access_token");
      const access_token_checked = localStorage.getItem("access_token_checked");

      if (access_token) {
        if (this.moreThanOneHourAgo(access_token_checked)) {
          this.axios
            .get(
              `https://api.untappd.com/v4/helpers/checkvalidtoken?utv=3.5.3&access_token=${access_token}`
            )
            .then((response) => {
              if (response.status === 200) {
                this.auth.token = access_token;
                localStorage.setItem("access_token_checked", Date.now());
              } else {
                localStorage.removeItem("access_token");
                localStorage.removeItem("access_token_checked");
                this.auth.token = null;
              }
            })
            .catch((error) => console.error(error));
        } else {
          this.auth.token = access_token;
        }
      }
    },
    getFlavorsById() {
      this.axios
        .get(
          `https://api.untappd.com/v4/beer/info/${this.beerId}?compact=enhanced&utv=3.5.3&access_token=${this.auth.token}`
        )
        .then((response) => {
          this.beerInfo = response.data.response.beer;
          this.loading = false;
        })
        .catch((error) => console.error(error));
    },
    onSubmit() {
      this.loading = true;

      if (!this.auth.token) {
        this.authUser()
          .then(() => this.getFlavorsById())
          .catch((error) => console.error(error));
      } else {
        this.getFlavorsById();
      }
    },
  },
};
</script>

<style scoped>
.auth {
  margin-bottom: 25px;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.output {
  margin-top: 50px;
}

.flavor {
  font-weight: bold;
}
</style>
