<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div ref="coverDiv" class="now-playing__cover">
        <img
          ref="trackAlbum"
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1
          ref="trackTitle"
          class="now-playing__track"
          v-text="player.trackTitle"
        ></h1>
        <h2
          ref="trackArtists"
          class="now-playing__artists"
          v-text="getTrackArtists"
        ></h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 ref="heading" class="now-playing__idle-heading">
        No music is playing 😔
      </h1>
      <h2 ref="secondary" class="now-playing__idle-secondary">
        💿 Hover your phone over an album cover to get started
      </h2>
    </div>
  </div>
</template>
<script src="node_modules/colorthief/dist/color-thief.umd.js"></script>

<script>
import ColorThief from '/node_modules/colorthief/dist/color-thief.mjs'

import { gsap } from 'gsap'
import props from '@/utils/props.js'

if ('paintWorklet' in CSS) {
  CSS.paintWorklet.addModule(
    'https://www.unpkg.com/css-houdini-squircle@0.1.3/squircle.min.js'
  )
}

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {}

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })

          return
        }

        data = await response.json()

        this.playerResponse = data
      } catch (error) {
        this.handleExpiredToken()

        data = this.getEmptyPlayer()
        this.playerData = data

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      // no image exists
      if (!this.player.trackAlbum?.image) {
        return
      }

      const colorThief = new ColorThief()
      const img = document.querySelector('img')
      const imageURL = img.src
      img.crossOrigin = 'Anonymous'
      document.querySelector('img').src = imageURL
      img.src = imageURL
      const handleFunc = this.handleAlbumPalette
      const handleFuncSecondary = this.handleAlbumPaletteSecondary

      let primaryCol
      let palletArray
      if (img.complete) {
        primaryCol = colorThief.getColor(img)
        handleFunc(primaryCol)
        palletArray = colorThief.getPalette(img)
        handleFuncSecondary(palletArray)
      } else {
        img.addEventListener('load', function() {
          primaryCol = colorThief.getColor(img)
          handleFunc(primaryCol)
          palletArray = colorThief.getPalette(img)
          handleFuncSecondary(palletArray)
        })
      }
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )
      document.documentElement.style.setProperty(
        '--color-text-alternate',
        this.colourPalette.text === '#000000' ? '#ffffff' : '#000000'
      )
      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()

        return
      }

      /**
       * Player is active, but user has paused.
       */
      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()

        return
      }

      /**
       * The newly fetched track is the same as our stored
       * one, we don't want to update the DOM yet.
       */
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        // this.playerData.progressPercent =
        //   this.playerResponse.progress_ms / this.playerResponse.item.duration_ms
        return
      }

      /**
       * Store the current active track.
       */
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        // progressPercent: 0,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    /**
     * Handle newly stored colour palette:
     * - Map data to readable format
     * - Get and store random colour combination.
     */
    handleAlbumPalette(palette) {
      const componentToHex = c => {
        var hex = c.toString(16)
        return hex.length == 1 ? '0' + hex : hex
      }
      const rgbToHex = (r, g, b) => {
        return '#' + componentToHex(r) + componentToHex(g) + componentToHex(b)
      }

      this.colourPalette = {
        text:
          palette[0] * 0.299 + palette[1] * 0.587 + palette[2] * 0.114 > 186
            ? '#000000'
            : '#ffffff',
        background: rgbToHex(palette[0], palette[1], palette[2])
      }
      gsap.to('.now-playing', {
        color: this.colourPalette.text,
        backgroundColor: this.colourPalette.background,
        duration: 1
      })

      this.setAppColours()
      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    handleAlbumPaletteSecondary(palette) {
      const componentToHex = c => {
        var hex = c.toString(16)
        return hex.length == 1 ? '0' + hex : hex
      }
      const rgbToHex = (r, g, b) => {
        return '#' + componentToHex(r) + componentToHex(g) + componentToHex(b)
      }
      const colourOne = palette[1]
      const hexColour = rgbToHex(colourOne[0], colourOne[1], colourOne[2])
      document.documentElement.style.setProperty(
        '--colour-background-shadow',
        hexColour
      )
      console.log('hexColour: ', hexColour)
      console.log(this.$refs.coverDiv)
      console.log(this.$refs.coverDiv.style.filter)
      console.log(hexColour)
      this.$refs.coverDiv.style.filter = `drop-shadow(0px 0px 5px ${hexColour})`
      console.log(this.$refs.coverDiv.style.filter)
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },
  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)
      if (this.$refs?.trackTitle)
        gsap.fromTo(
          [
            this.$refs?.trackTitle,
            this.$refs?.trackArtists
            // this.$refs?.progressBar
          ],
          {
            opacity: 0,
            stagger: 0.5,
            scale: 0.95
          },
          {
            opacity: 1,
            stagger: 0.5,
            scale: 1,
            duration: 1
          }
        )

      gsap.to('.now-playing', {
        color: this.colourPalette.text,
        backgroundColor: this.colourPalette.background,
        duration: 1
      })
      this.getAlbumColours()

      this.$nextTick(() => {
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
