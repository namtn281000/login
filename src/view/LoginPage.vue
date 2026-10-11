<template>
  <div class="login-page">
    <!-- Background: full-page globe -->
    <earth-globe
      class="bg-globe"
      :scale="globeScale"
      :center-y="globeCenterY"
      :clear-half-width="isStacked ? 0 : 218"
      @quality-change="onQuality"
    />

    <!-- Both sides: animated HUD panels (wide screens only) -->
    <side-hud v-if="showSides" side="left" :lite="lite" />
    <side-hud v-if="showSides" side="right" :lite="lite" />

    <!-- Centre: sign-in card on top of the globe -->
    <main class="stage">
      <section class="card" :class="{ 'card--lite': solidCard }">
        <header class="brand">
          <svg viewBox="0 0 32 32" aria-hidden="true">
            <circle class="logo-core" cx="16" cy="16" r="8.5" />
            <ellipse class="logo-orbit" cx="16" cy="16" rx="14" ry="5.5" transform="rotate(-24 16 16)" />
            <circle class="logo-sat" cx="27.4" cy="10.8" r="2.2" />
          </svg>
          <span>Meridian</span>
        </header>

        <h1 class="card__title">Welcome back</h1>
        <p class="card__subtitle">Sign in to continue to your workspace.</p>

        <v-form ref="form" v-model="valid" novalidate @submit.prevent="onSubmit">
          <div class="field">
            <label class="field__label" for="username">Username</label>
            <v-text-field
              id="username"
              v-model="form.username"
              :rules="rules.username"
              class="field__input"
              name="username"
              autocomplete="username"
              autocapitalize="none"
              spellcheck="false"
              placeholder="Enter your username"
              prepend-inner-icon="mdi-account-outline"
              color="#5CCBFF"
              hide-details="auto"
              outlined
              dark
            />
          </div>

          <div class="field">
            <div class="field__row">
              <label class="field__label" for="password">Password</label>
              <button type="button" class="link" @click="onForgot">Forgot password?</button>
            </div>
            <v-text-field
              id="password"
              v-model="form.password"
              :rules="rules.password"
              :type="passwordType"
              :append-icon="passwordIcon"
              class="field__input"
              name="password"
              autocomplete="current-password"
              placeholder="Enter your password"
              prepend-inner-icon="mdi-lock-outline"
              color="#5CCBFF"
              hide-details="auto"
              outlined
              dark
              @click:append="showPassword = !showPassword"
            />
          </div>

          <v-btn
            type="submit"
            class="btn-signin"
            :loading="loading"
            block
            depressed
            x-large
          >
            Sign in
          </v-btn>

          <p class="status" :class="{ 'status--error': statusIsError }" role="status">{{ status }}</p>
        </v-form>

        <footer class="foot">&copy; 2026 Meridian Systems</footer>
      </section>
    </main>
  </div>
</template>

<script>
import EarthGlobe from '../components/EarthGlobe.vue'
import SideHud from '../components/SideHub.vue'

export default {
  name: 'LoginPage',

  components: { EarthGlobe, SideHud },

  data() {
    return {
      valid: true,
      loading: false,
      showPassword: false,
      status: '',
      statusIsError: false,
      form: {
        username: '',
        password: ''
      },
      rules: {
        username: [(v) => !!(v && v.trim()) || 'Enter your username.'],
        password: [(v) => !!v || 'Enter your password.']
      },
      // layout state, kept in sync with the media queries below
      showSides: false,
      isStacked: false,
      // the globe's current quality level (0 = full); the card and side panels follow it
      quality: 0
    }
  },

  computed: {
    passwordType() {
      return this.showPassword ? 'text' : 'password'
    },
    passwordIcon() {
      return this.showPassword ? 'mdi-eye-off-outline' : 'mdi-eye-outline'
    },
    // Level 1: the card swaps its live blur for a solid fill. Level 2: the side panels stop animating.
    solidCard() {
      return this.quality >= 1
    },
    lite() {
      return this.quality >= 2
    },
    // On narrow screens the globe sits in the upper part of the screen and the card below it;
    // otherwise the globe is centred behind the card and drawn larger than the card.
    globeScale() {
      return this.isStacked ? 1 : 1.3
    },
    globeCenterY() {
      return this.isStacked ? 0.2 : 0.5
    }
  },

  watch: {
    // Clear the last message as soon as the user edits a field
    'form.username'() {
      this.status = ''
    },
    'form.password'() {
      this.status = ''
    }
  },

  created() {
    this.mqSides = window.matchMedia('(min-width: 1100px)')
    this.mqStacked = window.matchMedia('(max-width: 860px)') // keep in sync with the CSS media query below
    this.syncLayout()
  },

  mounted() {
    const queries = [this.mqSides, this.mqStacked]
    queries.forEach((mq) => {
      if (mq.addEventListener) mq.addEventListener('change', this.syncLayout)
      else mq.addListener(this.syncLayout)
    })
  },

  beforeDestroy() {
    const queries = [this.mqSides, this.mqStacked]
    queries.forEach((mq) => {
      if (mq.removeEventListener) mq.removeEventListener('change', this.syncLayout)
      else mq.removeListener(this.syncLayout)
    })
  },

  methods: {
    syncLayout() {
      this.showSides = this.mqSides.matches
      this.isStacked = this.mqStacked.matches
    },

    onQuality(level) {
      this.quality = level
    },

    async onSubmit() {
      if (this.loading || !this.$refs.form.validate()) return

      this.loading = true
      this.status = ''
      this.statusIsError = false
      try {
        await this.login({ username: this.form.username.trim(), password: this.form.password })
        // TODO: navigate after a successful login, e.g.
        // this.$router.push(this.$route.query.redirect || '/')
        this.status = 'Demo form: connect login() to your authentication API.'
      } catch (err) {
        this.statusIsError = true
        this.status = (err && err.message) || 'Unable to sign in. Try again.'
      } finally {
        this.loading = false
      }
    },

    // TODO: replace with the real call, e.g.
    // return this.$store.dispatch('auth/login', payload)
    // return this.$http.post('/auth/login', payload)
    login(payload) {
      return new Promise((resolve) => setTimeout(() => resolve(payload), 1400))
    },

    onForgot() {
      this.$emit('forgot-password')
      this.statusIsError = false
      this.status = 'Password reset is not connected in this demo.'
    }
  }
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500&family=Manrope:wght@400;500;600;700&family=Sora:wght@500;600&display=swap');

.login-page {
  --bg: #050a12;
  --field: rgba(10, 20, 36, 0.72);
  --line: rgba(150, 185, 230, 0.16);
  --line-strong: rgba(150, 185, 230, 0.34);
  --fg: #e9f0fa;
  --label: #c4d2e6;
  --muted: #8d9fb9;
  --faint: #70829f;
  --accent: #5ccbff;
  --accent-ink: #04121f;
  --signal: #ffb454;
  --danger: #ff7a8a;
  --font-display: 'Sora', 'Segoe UI', system-ui, sans-serif;
  --font-body: 'Manrope', 'Segoe UI', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', ui-monospace, 'SFMono-Regular', Menlo, Consolas, monospace;

  position: relative;
  box-sizing: border-box;
  min-height: 100vh;
  min-height: 100dvh;
  overflow: hidden;
  background: var(--bg);
  color: var(--fg);
  font: 400 15px/1.5 var(--font-body);
  -webkit-font-smoothing: antialiased;
}
.login-page *,
.login-page *::before,
.login-page *::after {
  box-sizing: border-box;
}

/* ---------- Layers: globe < side panels < card ---------- */
.login-page > .bg-globe {
  position: absolute;
  inset: 0;
  z-index: 0;
}

.stage {
  position: relative;
  z-index: 2;
  min-height: 100vh;
  min-height: 100dvh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px 16px;
}

/* ---------- Card ---------- */
.card {
  position: relative;
  width: 100%;
  max-width: 392px;
  padding: 28px 28px 18px;
  border: 1px solid var(--line-strong);
  border-radius: 24px;
  /* translucent on purpose: the globe shows through the card */
  background: linear-gradient(180deg, rgba(12, 22, 40, 0.42), rgba(5, 10, 20, 0.5));
  -webkit-backdrop-filter: blur(7px) saturate(125%);
  backdrop-filter: blur(7px) saturate(125%);
  box-shadow:
    0 30px 80px -20px rgba(0, 0, 0, 0.7),
    0 0 70px -22px rgba(92, 203, 255, 0.28),
    inset 0 1px 0 rgba(255, 255, 255, 0.06);
}
/* glowing top edge */
.card::before {
  content: '';
  position: absolute;
  top: -1px;
  left: 14%;
  right: 14%;
  height: 1px;
  background: linear-gradient(90deg, transparent, rgba(130, 225, 255, 0.85), transparent);
  pointer-events: none;
}
/* Globe dropped its quality: the live blur over a moving canvas is costly, so swap it for a solid fill */
.card--lite {
  -webkit-backdrop-filter: none;
  backdrop-filter: none;
  background: linear-gradient(180deg, rgba(12, 22, 40, 0.92), rgba(6, 12, 22, 0.94));
}

.brand {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  margin-bottom: 16px;
  font: 600 18px/1 var(--font-display);
  letter-spacing: -0.01em;
}
.brand svg {
  width: 30px;
  height: 30px;
  flex: none;
}
.logo-core { fill: none; stroke: var(--accent); stroke-width: 2; }
.logo-orbit { fill: none; stroke: var(--fg); stroke-opacity: 0.6; stroke-width: 1.5; }
.logo-sat { fill: var(--signal); }

.card__title {
  margin: 0 0 8px;
  text-align: center;
  font: 600 clamp(26px, 3vw, 32px) / 1.15 var(--font-display);
  letter-spacing: -0.025em;
  text-wrap: balance;
  text-shadow: 0 1px 14px rgba(3, 8, 16, 0.7);
}
.card__subtitle {
  margin: 0 0 22px;
  text-align: center;
  color: #a3b3cb;
  text-shadow: 0 1px 12px rgba(3, 8, 16, 0.7);
}

.field {
  display: grid;
  gap: 8px;
  margin-bottom: 16px;
}
.field__row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  gap: 12px;
}
.field__label {
  font: 600 13px/1.2 var(--font-body);
  color: var(--label);
  text-shadow: 0 1px 10px rgba(3, 8, 16, 0.7);
}

.link {
  padding: 0;
  border: 0;
  border-radius: 4px;
  background: none;
  color: var(--accent);
  font: 600 13px/1.2 var(--font-body);
  cursor: pointer;
}
.link:hover { text-decoration: underline; }
.link:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; }

/* ---------- Vuetify 2 overrides (v-text-field, outlined, dark) ---------- */
.login-page .field__input.v-text-field {
  margin: 0;
  padding: 0;
}
.login-page .field__input ::v-deep .v-input__slot {
  min-height: 50px;
  margin: 0;
  padding: 0 4px 0 12px;
  border-radius: 12px;
  background: var(--field);
  transition: box-shadow 0.2s;
}
.login-page .field__input ::v-deep fieldset {
  border-radius: 12px;
  border-width: 1px;
  color: var(--line) !important;
  transition: color 0.2s;
}
.login-page .field__input ::v-deep .v-input__slot:hover fieldset {
  color: var(--line-strong) !important;
}
.login-page .field__input.v-input--is-focused ::v-deep fieldset {
  border-width: 1px;
  color: var(--accent) !important;
}
.login-page .field__input.v-input--is-focused ::v-deep .v-input__slot {
  box-shadow: 0 0 0 4px rgba(92, 203, 255, 0.16);
}
.login-page .field__input.error--text ::v-deep fieldset {
  color: var(--danger) !important;
}
.login-page .field__input.error--text.v-input--is-focused ::v-deep .v-input__slot {
  box-shadow: 0 0 0 4px rgba(255, 122, 138, 0.16);
}
.login-page .field__input ::v-deep input {
  padding: 0;
  color: var(--fg);
  font: 500 15px var(--font-body);
}
.login-page .field__input ::v-deep input::placeholder {
  color: var(--faint) !important;
  opacity: 1;
}
.login-page .field__input ::v-deep .v-icon {
  color: var(--faint) !important;
  font-size: 20px;
}
.login-page .field__input.v-input--is-focused ::v-deep .v-input__prepend-inner .v-icon {
  color: var(--accent) !important;
}
.login-page .field__input ::v-deep .v-input__prepend-inner {
  margin-top: 0;
  padding-right: 10px;
  align-self: center;
}
.login-page .field__input ::v-deep .v-input__append-inner {
  margin-top: 0;
  align-self: center;
}
.login-page .field__input ::v-deep .v-messages__message {
  color: var(--danger);
  font-size: 13px;
  line-height: 1.4;
}
.login-page .field__input ::v-deep .v-text-field__details {
  margin-top: 8px;
  margin-bottom: 0;
}
/* keep browser autofill dark */
.login-page .field__input ::v-deep input:-webkit-autofill {
  -webkit-text-fill-color: var(--fg);
  caret-color: var(--fg);
  transition: background-color 99999s ease-out;
}

/* ---------- Submit button (v-btn) ---------- */
.login-page .btn-signin.v-btn.v-btn--has-bg {
  margin-top: 8px;
  height: 52px !important;
  border-radius: 12px;
  background: linear-gradient(135deg, #7ad8ff 0%, #4fc3ff 48%, #3ce0c8 100%);
  box-shadow: 0 12px 32px -12px rgba(76, 200, 255, 0.6), inset 0 1px 0 rgba(255, 255, 255, 0.4) !important;
  color: var(--accent-ink);
  font: 700 15px var(--font-body);
  letter-spacing: 0.01em;
  text-transform: none;
  transition: transform 0.15s, box-shadow 0.2s, filter 0.2s;
}
.login-page .btn-signin.v-btn.v-btn--has-bg:hover {
  filter: brightness(1.07);
}
.login-page .btn-signin.v-btn.v-btn--has-bg:active {
  transform: translateY(1px);
}
.login-page .btn-signin.v-btn--loading {
  filter: saturate(0.75) brightness(0.92);
}
.login-page .btn-signin ::v-deep .v-btn__loader {
  color: var(--accent-ink);
}

.status {
  min-height: 1.5em;
  margin: 16px 0 0;
  text-align: center;
  color: var(--muted);
  font-size: 13px;
}
.status--error { color: var(--danger); }

.foot {
  margin-top: 10px;
  padding-top: 12px;
  border-top: 1px solid var(--line);
  text-align: center;
  color: var(--faint);
  font: 400 12px/1.4 var(--font-mono);
  letter-spacing: 0.04em;
}

/* ---------- Narrow screens: globe in the upper part, card below it ---------- */
@media (max-width: 860px) {
  .stage {
    padding-top: clamp(250px, 34vh, 340px);
    padding-bottom: 52px; /* keep clear of the globe's bottom HUD row */
  }
}
@media (max-width: 640px) {
  .card {
    padding: 26px 20px 18px;
  }
}
</style>