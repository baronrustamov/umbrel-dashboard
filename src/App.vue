<template>
  <div id="app">
    <!-- insert wallpaper in the root component since it's used across all views -->
    <div v-if="wallpaper" class="wallpaper"
      :class="wallpaperClassList"
      :style="{backgroundImage: `url('/wallpapers/${wallpaper}')`}"
    ></div>
    <transition name="loading" mode>
      <div v-if="isIframe">
        <div class="d-flex flex-column align-items-center justify-content-center min-vh100 p-2">
          <logo class="mb-5 logo" />
          <span class="text-muted w-75 text-center">
            <small>For security reasons Портал cannot be embedded in an iframe.</small>
          </span>
        </div>
      </div>
      <loading v-else-if="updating" :progress="updateStatus.progress">
        <div class="text-center">
          <small class="text-white d-block">{{`${updateStatus.description}...`}}</small>
          <b-alert class="alert-system" variant="glass" show>
            <div class="d-flex align-items-center">
              <small>Please do not refresh this page or turn off your Портал while the update is in progress</small>
            </div>
          </b-alert>
        </div>
      </loading>
      <shutdown
        v-else-if="hasShutdown || shuttingDown || rebooting"
        :hasShutdown="hasShutdown"
        :shuttingDown="shuttingDown"
        :rebooting="rebooting"
      >
        <div class="text-center" v-if="shuttingDown || rebooting">
          <b-alert class="alert-system" variant="glass" show>
            <div class="d-flex align-items-center">
              <small>Please do not refresh this page or turn off your Портал while it is {{ shuttingDown ? 'shutting down' : 'rebooting'}}</small>
            </div>
          </b-alert>
        </div>
      </shutdown>
      <loading v-else-if="loading" :progress="loadingProgress">
      </loading>
      <!-- component matched by the route will render here -->
      <router-view v-else></router-view>
    </transition>

    <!-- Preload dock icons -->
    <img v-for="image in ['home', 'settings', 'app-store', 'sun', 'moon', 'logout']" :key="image" class="d-none" :src="require(`@/assets/dock/${image}.png`)" alt="" draggable="false">
  </div>
</template>

<style lang="scss">
@import "@/global-styles/design-system.scss";
</style>

<script>
import { mapState } from "vuex";
import delay from "@/helpers/delay";
import Shutdown from "@/components/Shutdown";
import Loading from "@/components/Loading";
import Logo from '@/components/Logo.vue';

export default {
  name: "App",
  data() {
    return {
      isIframe: (window.self !== window.top),
      loading: true,
      loadingProgress: 0,
      loadingPollInProgress: false
    };
  },
  computed: {
    ...mapState({
      wallpaper: state => state.user.wallpaper,
      hasShutdown: state => state.system.hasShutdown,
      shuttingDown: state => state.system.shuttingDown,
      rebooting: state => state.system.rebooting,
      isManagerApiOperational: state => state.system.managerApi.operational,
      isApiOperational: state => state.system.api.operational,
      updateStatus: state => state.system.updateStatus,
      jwt: state => state.user.jwt
    }),
    updating() {
      return this.updateStatus.state === "installing";
    },
    wallpaperClassList() {
      const classList = [];
      if (!window.CSS.supports('backdrop-filter', 'blur(0)')) {
        classList.push("wallpaper-no-backdrop-blur");
      }
      if (this.loading || this.isIframe || this.shuttingDown || this.rebooting || this.updating || this.hasShutdown) {
        classList.push("wallpaper-blur wallpaper-slight-dim wallpaper-zoom-in");
        return classList;
      }
      if (this.$route.meta && this.$route.meta.wallpaperClass) {
        classList.push(this.$route.meta.wallpaperClass);
      }
      return classList.join(" ");
    }
  },
  methods: {
    //TODO: move this to the specific layout that needs this 100vh fix
    updateViewPortHeightCSS() {
      return document.documentElement.style.setProperty(
        "--vh100",
        `${window.innerHeight}px`
      );
    },
    async getLoadingStatus() {
      // Skip if previous poll in progress or if system is updating
      if (this.loadingPollInProgress || this.updating) {
        return;
      }

      this.loadingPollInProgress = true;

      // First check if manager api is up and get the wallpaper
      if (this.loadingProgress <= 20) {
        this.loadingProgress = 20;
        await this.$store.dispatch("system/getManagerApi");
        if (!this.isManagerApiOperational) {
          this.loading = true;
          this.loadingPollInProgress = false;
          return;
        }
      }

      // Then trigger auth check
      if (this.loadingProgress <= 70 && this.jwt) {
        this.loadingProgress = 70;
        try {
          await this.$store.dispatch("user/getInfo");
          // load apps
          await this.$store.dispatch("apps/getInstalledApps");
        } catch (error) {
          // it will error if jwt has expired and automatically
          // redirect the user to login page
        }
      }

      this.loadingProgress = 100;
      this.loadingPollInProgress = false;

      // Add slight delay so the progress bar makes
      // it to 100% before disappearing
      setTimeout(() => (this.loading = false), 300);
    }
  },
  created() {
    // check if system is updating
    this.$store.dispatch("system/getUpdateStatus");

    // get light/dark mode preference
    this.$store.dispatch("system/getDarkMode");

    // get wallpaper
    this.$store.dispatch("user/getWallpaper");
    
    // for 100vh consistency
    this.updateViewPortHeightCSS();
    window.addEventListener("resize", this.updateViewPortHeightCSS);
  },
  watch: {
    loading: {
      handler: function(isLoading) {
        window.clearInterval(this.loadingInterval);
        //if loading, check loading status every two seconds
        if (isLoading) {
          this.loadingInterval = window.setInterval(
            this.getLoadingStatus,
            2000
          );
        } else {
          //else check every 20s
          this.loadingInterval = window.setInterval(
            this.getLoadingStatus,
            20000
          );
        }
      },
      immediate: true
    },
    updating: {
      handler: function(isUpdating, wasUpdating) {
        window.clearInterval(this.updateStatusInterval);
        // if updating, check loading status every two seconds
        if (isUpdating) {
          this.updateStatusInterval = window.setInterval(() => {
            this.$store.dispatch("system/getUpdateStatus");
          }, 2 * 1000);
        } else {
          //else check every minute
          this.updateStatusInterval = window.setInterval(() => {
            this.$store.dispatch("system/getUpdateStatus");
          }, 60 * 1000);

          // if it just finished updating, then show success/failure toast
          if (wasUpdating) {
            const toastOptions = {
              title: "Update successful",
              autoHideDelay: 2000,
              variant: "success",
              solid: true,
              toaster: "b-toaster-bottom-right"
            };

            if (this.updateStatus.state === "failed") {
              toastOptions.title = "Update failed";
              toastOptions.variant = "danger";
            }

            this.$bvToast.toast(this.updateStatus.description, toastOptions);

            //refresh window to fetch latest code of dashboard
            delay(2000).then(() => {
              window.location.reload(true);
            });
          }
        }
      },
      immediate: true
    }
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.updateViewPortHeightCSS);
    window.clearInterval(this.loadingInterval);
    window.clearInterval(this.updateStatusInterval);
  },
  components: {
    Loading,
    Shutdown,
    Logo,
  }
};
</script>

<style lang="scss" scoped>
// Loading transitions

.loading-enter-active,
.loading-leave-active {
  transition: opacity 0.4s ease;
}
.loading-enter {
  opacity: 0;
}
.loading-enter-to {
  opacity: 1;
}
.loading-leave {
  opacity: 1;
}
.loading-leave-to {
  opacity: 0;
}

.alert-system {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translate3d(-50%, 0, 0);
}

.wallpaper {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: var(--vh100);
  z-index: -1;
  background-size: cover;
  background-position: center;
  will-change: transform;
  transition: background-image 0.3s, filter 1s, transform 1.5s ease;
  &:after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    backdrop-filter: blur(0);
    transition: backdrop-filter 0.4s ease;
  }
  &.wallpaper-blur {
    filter: blur(0);
    &:after {
      backdrop-filter: blur(10px);
    }
  }
  &.wallpaper-slight-dim {
    filter: brightness(0.9);
  }
  &.wallpaper-content-open {
    filter: brightness(var(--wallpaper-brightness));
  }
  &.wallpaper-zoom-in {
    transform: scale3d(1.25, 1.25, 1.25);
  }

  // use blur filter if backdrop blur isn't supported.
  // we check this via JS and not CSS @supports media query because
  // vue auto-prefixes backdrop-blur with -webkit vendor prefix
  // which tests positive for Safari, while we want this behavior off
  // for Safari due to artificacts on edges. that's the opposite of
  // Chrome, which has artificacts on edges using blur filter
  // but not backdrop-blur
  &.wallpaper-no-backdrop-blur {
    &:after {
      display: none !important;
    }
    &.wallpaper-blur {
      filter: blur(12px);
      &.wallpaper-slight-dim {
        filter: blur(12px) brightness(0.9);
      }
    }
  }
}
</style>
