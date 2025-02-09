<script lang="ts">
  import { Component, Prop, Vue, Watch } from 'vue-property-decorator'
  import { namespace } from 'vuex-class'
  import SearchBox from '@/components/SearchBox.vue'
  import SignInButton from '@/components/SignInButton.vue'
  import _ from 'lodash'

  const uiModule = namespace('ui')
  const authenticationModule = namespace('authentication')
  const characterModule = namespace('character')

  @Component({
    components: {
      SearchBox,
      SignInButton
    }
  })
  export default class MainToolbar extends Vue {
    @uiModule.State isSideBarOpen!: boolean
    @uiModule.State isDarkSide!: boolean
    @uiModule.Action updateSideBar!: (value: boolean) => void
    @authenticationModule.Getter isLoggedIn!: boolean
    @authenticationModule.Action setAccessToken!: (accessToken?: string) => Promise<any>
    @authenticationModule.State isAuthLoading!: boolean
    @authenticationModule.Action setIsAuthLoading!: (isAuthLoading: boolean) => Promise<any>
    @characterModule.Action clearLocalCharacters!: () => Promise<any>

    isSearchOpen = false
    toolbarLoading = true

    async mounted () {
      this.toolbarLoading = false
      if (this.isAuthLoading) {
        setTimeout(() => {
          this.setIsAuthLoading(false)
        }, 5000)
      }
    }

    @Watch('$route')
    resetSearch () {
      this.isSearchOpen = false
    }

    get routes () {
      return [
        {
          to: '/',
          title: 'Home'
        },
        {
          to: '/rules',
          title: 'Rules',
          nested: [
            { to: '/phb', title: 'Player\'s Handbook' },
            { to: '/snv', title: 'Scum and Villainy' },
            { to: '/sotg', title: 'Starships of the Galaxy' },
            { to: '/wh', title: 'Wretched Hives' },
            { to: '/variantRules', title: 'Variant Rules' },
            { to: '/expandedContent', title: 'Expanded Content' }
          ]
        },
        {
          to: '/characters',
          title: 'Characters',
          nested: [
            { to: '/species', title: 'Species' },
            { to: '/classes', title: 'Classes' },
            { to: '/archetypes', title: 'Archetypes' },
            { to: '/backgrounds', title: 'Backgrounds' },
            { to: '/feats', title: 'Feats' },
            { to: '/customizationOptions', title: 'Customization Options' },
            { to: '/forcePowers', title: 'Force Powers' },
            { to: '/techPowers', title: 'Tech Powers' },
            { to: '/maneuvers', title: 'Maneuvers' }
          ]
        },
        {
          to: '/loot',
          title: 'Loot',
          nested: [
            { to: '/armor', title: 'Armor' },
            { to: '/weapons', title: 'Weapons' },
            { to: '/adventuringGear', title: 'Adventuring Gear' },
            { to: '/enhancedItems', title: 'Enhanced Items' }
          ]
        },
        {
          to: '/starships',
          title: 'Starships',
          nested: [
            { to: '/deployments', title: 'Deployments' },
            { to: '/ventures', title: 'Ventures' },
            { to: '/modifications', title: 'Modifications' },
            { to: '/equipment', title: 'Equipment' },
            { to: '/weapons', title: 'Weapons' }
          ]
        },
        {
          to: '/tools',
          title: 'Tools',
          nested: [
            { to: '/mycharacters', title: 'Character Creator' }
          ]
        },
        {
          to: '/assets',
          title: 'Assets'
        },
        {
          to: '/credits',
          title: 'Credits'
        }
      ]
    }

    get isPageWithNavigation () {
      return ['phb', 'wh', 'sotg', 'snv'].some(page => this.$route.path.includes(page))
    }

    get darkColor () {
      return this.$vuetify.theme.dark ? '' : 'primary'
    }

    buildComponentProps (to: string, nested: { to: string, title: string }[]) {
      return nested && nested.length ? {
        is: 'v-menu',
        'open-on-hover': true,
        'offset-y': true,
        attach: true
      } : {
        is: 'v-btn',
        text: true,
        color: this.darkColor,
        to
      }
    }

    handleSideIconClick () {
      this.updateSideBar(true)
    }

    logOut () {
      Promise.all([
        this.setAccessToken(),
        this.clearLocalCharacters()
      ]).then(() => {
        Vue.prototype.$msal && Vue.prototype.$msal.logoutRedirect({
          postLogoutRedirectUri: window.location.pathname + '#logged-out'
        })
      })
    }
  }
</script>

<template lang="pug">
  v-app-bar(app, clipped-left, :color="$vuetify.theme.dark ? 'secondary' : undefined").d-print-none
    v-app-bar-nav-icon(v-if="isPageWithNavigation", @click="handleSideIconClick").hidden-md-and-up
    v-toolbar-title
      router-link(to="/")
        v-img(:src="require('@/assets/sw5e-logo.png')", width="100px")
    v-spacer
    SearchBox(v-if="isSearchOpen").ml-2
    v-toolbar-items.hidden-sm-and-down
      template(v-if="!isSearchOpen")
        component(v-for="({ to, title, nested }) in routes", :key="title", v-bind="buildComponentProps(to, nested)")
          template(v-if="nested && nested.length", v-slot:activator="{ on }")
            v-btn(text, :color="darkColor", v-on="on", :to="to") {{ title }}
              v-icon.pl-2 fa-caret-down
          v-list(v-for="nestedRoute in nested", :key="nestedRoute.title", dense)
            v-list-item(:to="to + nestedRoute.to")
              v-list-item-title {{ nestedRoute.title }}
          template(v-if="!nested || !nested.length") {{ title }}
        v-btn(v-if="(isAuthLoading || toolbarLoading)", :color="darkColor")
          v-progress-circular(indeterminate, color="white", size="15")
        v-menu(v-else-if="isLoggedIn", offset-y)
          template(v-slot:activator="{ on }")
            v-btn(text, :color="darkColor", v-on="on")
              v-icon(:color="darkColor") fa-user
          v-list(dense)
            v-list-item(to="/profile")
              v-list-item-title Profile
            v-list-item(@click="logOut")
              v-list-item-title Logout
        SignInButton(v-else) Login
      v-btn(icon, @click="isSearchOpen = !isSearchOpen")
        v-icon(:color="darkColor") {{ isSearchOpen ? 'fa-times' : 'fa-search' }}
    v-toolbar-items.hidden-md-and-up
      v-btn(icon, @click="isSearchOpen = !isSearchOpen")
        v-icon {{ isSearchOpen ? 'fa-times' : 'fa-search' }}
      v-btn(v-if="(isAuthLoading || toolbarLoading)", :color="darkColor")
        v-progress-circular(indeterminate, color="white", size="15")
      v-menu(v-else-if="isLoggedIn", offset-y)
        template(v-slot:activator="{ on }")
          v-btn(text, :color="darkColor", v-on="on")
            v-icon(:color="darkColor") fa-user
        v-list(dense)
          v-list-item(to="/profile")
            v-list-item-title Profile
          v-list-item(@click="logOut")
            v-list-item-title Logout
      SignInButton(v-else) Login
      v-menu(v-if="!isSearchOpen", bottom, left, offset-y, attach)
        template(v-slot:activator="{ on }")
          v-btn(icon, v-on="on")
            v-icon fa-ellipsis-v
        v-list
          v-list-item(v-for="{to, title} in routes", :key="title", :to="to")
            v-list-item-title {{ title }}
          v-list-item(to="/searchResults")
            v-icon.pr-2 fa-search
            v-list-item-title Search
</template>
