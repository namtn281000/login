<template>
  <v-app>
    <login-page></login-page>
  </v-app>
</template>

<script>
import LoginPage from "./view/LoginPage.vue";
export default {
  name: "App",

  components: {
    LoginPage,
  },

  data: () => ({
    //
  }),
};
</script>
