<script> // eslint-disable-line vue/multi-word-component-names
/**
 * TODO: Extend or create an alternative leaderboard compatible with teams (humans/ogres)
 * TODO: This leaderboard is not only shown on the league url but also the ladder url.
 */
import utils from 'core/utils'
const d3 = require('d3/d3.js')

export default Vue.extend({
  name: 'LeaderboardComponent',
  props: {
    scoreType: {
      type: String,
      default: 'arena'
    },
    league: {
      type: Object,
      default: null
    },
    leagueType: {
      type: String,
      default: 'clan'
    },
    course: {
      type: Object,
      default: null
    },
    level: {
      type: Object,
      default: () => {}
    },
    playerCount: {
      type: Number,
      default: 0
    },
    clanId: {
      type: String,
      default: '_global'
    },
    title: {
      type: String,
      default: ''
    },
    tableTitles: {
      type: Array,
      default () {
        return []
      }
    }
  },
  data () {
    return {
      selectedRow: [],
      ageFilter: false,
      dateBeforeSep: new Date() < new Date('2022-9-1')
    }
  },
  computed: {
    ageBrackets () {
      let brackets = utils.ageBrackets
      if (this.$store.state.features.china) {
        brackets = utils.ageBracketsChina
      }
      return brackets.map((b) => {
        return {
          name: this.getAgeBracket(b.slug),
          slug: b.slug
        }
      })
    }
  },
  mounted () {
    if (this.scoreType === 'tournament') {
      // We don't need to get the histogram when it's a tournament, sicne we won't show it, and those data are slow to fetch.
      return
    }
    const histogramWrapper = $('#histogram-display-humans')
    let histogramData = null
    let url = `/db/level/${this.level.get('original')}/rankings-histogram?team=humans&levelSlug=${this.level.get('slug')}`
    if (this.league) {
      url += '&leagues.leagueID=' + this.league.id
    }
    $.when(
      $.get(url, (data) => { histogramData = data })
    ).then(() => this.generateHistogram(histogramWrapper, histogramData, 'humans'))
  },
  methods: {
    loadMore () {
      this.$emit('load-more')
    },
    generateHistogram (histogramElement, histogramData, teamName) {
      console.log('generate hist 1')
      // renders twice, hack fix
      if ($('#' + histogramElement.attr('id')).has('svg').length) {
        return
      }
      if (!histogramData.length) {
        return histogramElement.hide()
      }

      histogramData = histogramData.map((d) => d * 100)
      const margin = {
        top: 20,
        right: 20,
        bottom: 30,
        left: 15
      }
      const width = 470 - margin.left - margin.right
      const height = 125 - margin.top - margin.bottom
      const axisFactor = 1000
      const minX = Math.floor(Math.min(...histogramData) / axisFactor) * axisFactor
      const maxX = Math.ceil(Math.max(...histogramData) / axisFactor) * axisFactor
      const x = d3.scale.linear().domain([minX, maxX]).range([0, width])
      const data = d3.layout.histogram().bins(x.ticks(20))(histogramData)
      const y = d3.scale.linear().domain([0, d3.max(data, (d) => d.y)]).range([height, 10])

      // create the x axis
      const xAxis = d3.svg.axis().scale(x).orient('bottom').ticks(5).outerTickSize(0)

      const svg = d3.select('#histogram-display-humans').append('svg')
        .attr('preserveAspectRatio', 'xMinYMin meet')
        .attr('viewBox', `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
        .append('g')
        .attr('transform', `translate(${margin.left}, ${margin.top})`)
      const barClass = 'humans-bar'

      const bar = svg.selectAll('.bar')
        .data(data)
        .enter().append('g')
        .attr('class', barClass)
        .attr('transform', (d) => `translate(${x(d.x)}, ${y(d.y)})`)

      bar.append('rect')
        .attr('x', 1)
        .attr('width', width / 20)
        .attr('height', (d) => height - y(d.y))
      if (this.session) {
        let playerScore = this.session.get('totalScore')
        if (this.league) {
          const league = window._.find(this.session.get('leagues'), { leagueID: this.league.id })
          playerScore = league ? +league.stats.totalScore : 10
        }
        playerScore = playerScore * 100

        const scorebar = svg.selectAll('.specialbar')
          .data([playerScore])
          .enter().append('g')
          .attr('class', 'specialbar')
          .attr('transform', `translate(${x(playerScore)}, 0)`)

        scorebar.append('rect')
          .attr('x', 1)
          .attr('width', 2)
          .attr('height', height)
      }
      const rankClass = 'rank-text humans-rank-text'

      const message = `${histogramData.length.toLocaleString()} players`
      /* if(@leaderboards[teamName].session)
       *   //# TODO: i18n for these messages
       *   if(this.league)
       *     //# TODO: fix server handler to properly fetch myRank with a leagueID
       *     message = "#{histogramData.length} players in league" */
      /* else if(@leaderboards[teamName].myRank <= histogramData.length) {
       *   message = "##{@leaderboards[teamName].myRank} of #{histogramData.length}"
       *   message += "+" if histogramData.length >= 100000
       * } */
      /* else if(@leaderboards[teamName].myRank is 'unknown')
       *   message = "#{if histogramData.length >= 100000 then '100,000+' else histogramData.length} players"
       * else
       *   message = 'Rank your session!' */
      svg.append('g')
        .append('text')
        .attr('class', rankClass)
        .attr('y', 0)
        .attr('text-anchor', 'end')
        .attr('x', width)
        .text(message)

      // Translate the x-axis up
      svg.append('g')
        .attr('class', 'x axis')
        .attr('transform', 'translate(0, ' + height + ')')
        .call(xAxis)

      histogramElement.show()
    },
    toggleAgeFilter () {
      this.ageFilter = !this.ageFilter
    },
    filterAge (slug) {
      this.$emit('filter-age', slug)
      this.ageFilter = false
    },
    computeClass (slug, item = '') {
      if (slug === 'name') {
        return { 'name-col-cell': 1, ai: /(Bronze|Silver|Gold|Platinum|Diamond) AI/.test(item) }
      }
      if (slug === 'team' || slug === 'clan') {
        return { capitalize: 1, 'clan-col-cell': 1 }
      }
      if (slug === 'language') {
        return { 'code-language-cell': 1 }
      }
    },
    computeStyle (item, index) {
      if (this.tableTitles[index].slug === 'language' && item) {
        return { 'background-image': `url(/images/common/code_languages/${item}_icon.png)` }
      }
    },
    getAgeBracket (item) {
      return $.i18n.t(`ladder.bracket_${(item || 'open').replace(/-/g, '_')}`)
    },

    getCountry (code) {
      return utils.countryCodeToFlagEmoji(code)
    },

    getCountryName (code) {
      return utils.countryCodeToName(code)
    },

    computeTitle (slug, item) {
      if (slug === 'country') {
        return this.getCountryName(item)
      } else {
        return ''
      }
    },
    computeBody  (slug, item) {
      if (slug === 'country') {
        return this.getCountry(item)
      } else if (slug === 'fight') {
        let url = `/play/level/${this.level.get('slug')}?team=humans&opponent=${item}`
        if (this.league) {
          if (this.leagueType === 'clan') {
            url += `&league=${this.league.id}`
          } else if (this.leagueType === 'course') {
            url += `&course=${this.course.id}&course-instance=${this.league.id}`
          }
        }
        return '<a href="' + url + '"><span>' + $.i18n.t('ladder.fight') + '</span></a>'
      } else {
        return item
      }
    },
    classForRow (row) {
      if (this.session) {
        if (row[0] === this.session.get('creator')) {
          return 'my-row'
        }
      }
      if (window.location.pathname === '/league' && row.fullName) {
        return 'student-row'
      }
      return ''
    },
    onClickUserRow (rank, slug, nearby = false) {
      if (slug !== 'fight') {
        this.$emit('click-player-name', rank, nearby)
      }
    },
    onClickSpectateCell (rank) {
      const index = this.selectedRow.indexOf(rank)
      if (index !== -1) {
        this.$delete(this.selectedRow, index)
      } else {
        this.selectedRow = this.selectedRow.concat([rank]).slice(-2)
      }
      this.$emit('spectate', this.selectedRow)
    },
    unlockEsports () {
      if (this.dateBeforeSep) {
        this.$emit('temp-unlock')
      } else {
        window.open('https://form.typeform.com/to/qXqgbubC', '_blank')
      }
    }
  }
})
</script>

<template lang="pug">
  .table-responsive(:class="{'col-md-7': scoreType=='arena'}")
    div(v-if="scoreType == 'arena'")
      div(:class="{hide: !showContactUs}" id="contact-us-info")
        if dateBeforeSep
          p.title {{ $t('league.esports_anonymous_changing') }}
        else
          p.title {{ $t('league.unlock_ai_league') }}
        .content
          img.img(src="/images/pages/play/ladder/esports_lock.png")
          .right-info
            a.unlock-button(@click="unlockEsports" style="margin-right: 0.6em;") {{ $t('league.esports_get_full_access') }}
            if dateBeforeSep
              p {{ $t('league.click_to_unlock_now') }}
            else
              p {{ $t('league.unlock_content_padding') }}
      div(id="histogram-display-humans", class="histogram-display", data-team-name='humans' :class="{hide: showContactUs}")
    table.table.table-bordered.table-condensed.table-hover.ladder-table
      thead
        tr
          th(colspan=13)
            span {{ title }}
            span &nbsp;
            span {{ $t('ladder.leaderboard') }}
            span(v-if="playerCount > 1")
              span  -&nbsp;
              span {{ playerCount.toLocaleString() }}
              span  players

        tr
          th(v-for="t in tableTitles" v-if="!(['creator', 'hide'].includes(t.slug))" :key="t.slug" :colspan="t.col" :class="computeClass(t.slug)")
            | {{ t.title }}
            span &nbsp;
            span.age-filter(v-if="t.slug == 'age'")
              .glyphicon.glyphicon-filter(@click="toggleAgeFilter")
              #age-filter(:class="{display: ageFilter}")
                .slug(v-for="bracket in ageBrackets" @click="filterAge(bracket.slug)")
                  span {{bracket.name}}

      tbody
        tr(v-for="row, rank in rankings" :key="rank" :class="classForRow(row)")
          template(v-if="row.type==='BLANK_ROW'")
            td(colspan=3) ...
          template(v-else)
            td(v-for="item, index in row" v-if="index > 0 && !(['fight', 'hide'].includes(tableTitles[index].slug))" :key="'' + rank + index" :colspan="tableTitles[index].col" :style="computeStyle(item, index)" :class="computeClass(tableTitles[index].slug, item)" :title="computeTitle(tableTitles[index].slug, item)" @click="onClickUserRow(rank, tableTitles[index].slug)") {{index != 1 ? computeBody(tableTitles[index].slug, item): '' }}
            td(colspan=1 v-if="tableTitles[row.length-1].slug == 'fight'" v-html="computeBody('fight', row[row.length-1])" @click="onClickUserRow(rank, 'fight')")

        tr(v-for="row, rank in playerRankings" :key="'player-'+rank" :class="classForRow(row)")
          template(v-if="row.type==='BLANK_ROW'")
            td(colspan=3) ...
          template(v-else)
            td(v-for="item, index in row" v-if="index > 0" :key="'player-' + rank + index" :colspan="tableTitles[index].col" :style="computeStyle(item, index)" :class="computeClass(tableTitles[index].slug, item)" :title="computeTitle(tableTitles[index].slug, item)" v-html="index != 1 ? computeBody(tableTitles[index].slug, item): ''" @click="onClickUserRow(rank, tableTitles[index].slug, true)")

    #load-more.btn.btn-sm(data-i18n='editor.more', @click="loadMore")
</template>

<style scoped lang="scss">
.ladder-table {
  /* background-color: #F2F2F2; */
  background-color: white;
}
.ladder-table td {
  padding: 1px 2px;
}

.ladder-table .code-language-cell {
  height: 16px;
  background-position: center;
  background-size: 16px;
  background-repeat: no-repeat;
  padding: 0 10px
}

.ladder-table tr {
  font-size: 14px;
  text-align: center;
}
.ladder-table tbody tr:hover td{
  background-color: #FFFFFF;
}

.ladder-table th {
  text-align: center;
}

.name-col-cell, .clan-col-cell {
  max-width: 170px;
  text-overflow: ellipsis;
  white-space: nowrap;
  overflow: hidden;
}

.capitalize {
  text-transform: capitalize;
}

.name-col-cell.ai {
  color: #3f44bf;
}

.my-row {
  background-color: #d1b147;
}

.student-row {
  background-color: #bcff16;
}

.age-filter {
  position: relative;

  .glyphicon {
    cursor: pointer;
  }

  #age-filter {
    position: absolute;
    display: none;
    background-color: #fcf8f2;
    border: 1px solid #ddd;
    border-radius: 5px;
    right: 0;
    width: 5em;

    &.display {
      display: block;
    }

    .slug {
      margin: 5px 10px;
      cursor: pointer;
    }
  }
}
</style>

<style lang="scss">
#histogram-display-humans {
  position: relative;
  width: 100%;
  padding-bottom: 28%;
  vertical-align: top;
  overflow: hidden;
  height: 130px;
  background-color: white;

  svg {
    overflow: visible;

    .humans-bar rect {
      shape-rendering: crispEdges;
    }

    .humans-bar text{
      fill: #fff;
    }

    .specialbar rect {
      fill: #555555;
    }

    .axis path , .axis line{
      fill: none;
      stroke: #555555;
      shape-rendering: crispEdges;
    }

    .humans-bar {
      fill: #bf3f3f;
      shape-rendering: crispEdges;
    }
    text {
      fill: #555555;
    }

    .rank-text {
      font-size: 15px;
      fill: #555555;
    }

    .humans-rank-text {
      fill: #bf3f3f;
    }

  }
}
#contact-us-info {
  position: relative;
  width: 100%;
  padding-bottom: 28%;
  vertical-align: top;
  overflow: hidden;
  /* height: 130px; */
  color: white;
  background: url(/images/pages/play/ladder/unlock_esports_background.png);
  background-size: 100%;
  font-size: 18px;
  font-weight: 600;
  padding: 12px 40px;
  text-align: center;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;

  .title {
    width: 400px
  }

  .content {
    width: 380px;
    display: flex;

    .img {
      height: 4em;
    }

    .right-info {
      display: flex;
      flex-direction: column;
      align-items: center;

      .unlock-button {
        width: fit-content;
        text-transform: uppercase;
        color: white;
        display: block;
        padding: 0.5em;
        background: linear-gradient(to left top , #DB36A4, #F1D521);
        border-radius: 2em;

        &:hover {
          text-decoration: none;
        }
      }

      p {
        margin: 5px 0;
        font-size: 14px;
      }
    }
  }
}
.hide {
  display: none;
}
</style>
