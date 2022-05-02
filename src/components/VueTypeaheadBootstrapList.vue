<template>
  <div class="list-group shadow" ref="suggestionList">
    <vue-typeahead-bootstrap-list-item
      v-for="(item, id) in matchedItems" :key="id"
      :active="isListItemActive(id)"
      :id="(isListItemActive(id)) ? `selected-option-${vbtUniqueId}` : false"
      :data="item.data"
      :html-text="highlight(item.text)"
      :highlighted-tags="highlightedMatchedTags(item.data)"
      role="option"
      :aria-selected="(isListItemActive(id)) ? 'true' : 'false'"
      :screen-reader-text="(item.screenReaderText) ? item.screenReaderText : item.text"
      :disabled="isDisabledItem(item)"
      :background-variant="backgroundVariant"
      :background-variant-resolver="backgroundVariantResolver"
      :text-variant="textVariant"
      @click.native="handleHit(item, $event)"
      v-on="$listeners"
    >
      <template v-if="$scopedSlots.suggestion" slot="suggestion" slot-scope="{ data, htmlText, highlightedTags }">
        <slot name="suggestion" v-bind="{ data, htmlText, highlightedTags }" />
      </template>
    </vue-typeahead-bootstrap-list-item>
  </div>
</template>

<script>
import VueTypeaheadBootstrapListItem from './VueTypeaheadBootstrapListItem.vue'
import clone from 'lodash/clone'
import includes from 'lodash/includes'
import isEmpty from 'lodash/isEmpty'
import reject from 'lodash/reject'
import reverse from 'lodash/reverse'
import findIndex from 'lodash/findIndex'
import map from 'lodash/map'
import reduce from 'lodash/reduce'
import some from 'lodash/some'

const BEFORE_LIST_INDEX = -1

function sanitize(text) {
  return text.replace(/</g, '&lt;').replace(/>/g, '&gt;')
}

function escapeRegExp(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

export default {
  name: 'VueTypeaheadBootstrapList',

  components: {
    VueTypeaheadBootstrapListItem
  },

  props: {
    data: {
      type: Array,
      required: true,
      validator: d => d instanceof Array
    },
    query: {
      type: String,
      default: ''
    },
    vbtUniqueId: {
      type: Number,
      required: true
    },
    backgroundVariant: {
      type: String
    },
    backgroundVariantResolver: {
      type: Function,
      default: (d) => null,
      validator: d => d instanceof Function
    },
    disableSort: {
      type: Boolean
    },
    textVariant: {
      type: String
    },
    maxMatches: {
      type: Number,
      default: 10
    },
    minMatchingChars: {
      type: Number,
      default: 2
    },
    disabledValues: {
      type: Array,
      default: () => []
    },
    showOnFocus: {
      type: Boolean,
      default: false
    },
    showAllResults: {
      type: Boolean,
      default: false
    },
    highlightClass: {
      type: String,
      default: 'vbt-matched-text'
    },
    tagsSearchField: {
      type: String
    },
    searchTermField: {
      type: String
    },
    searchStringToken: {
      type: String
    }
  },

  created() {
    this.$parent.$on('input', this.resetActiveListItem)
    this.$parent.$on('keyup', this.handleParentInputKeyup)
  },
  data() {
    return {
      activeListItem: -1
    }
  },

  computed: {
    highlight() {
      return text => {
        text = sanitize(text)
        if (this.query.length === 0) {
          return text
        }

        const re = new RegExp(this.escapedQuery, 'gi')
        return text.replace(re, `<span class='${this.highlightClass}'>$&</span>`)
      }
    },

    highlightedMatchedTags() {
      return data => {
        let allSearchTerms = this.showAllResults ? [''] : this.escapedQuery.split(this.searchStringToken);

        if (!isEmpty(allSearchTerms)) {
          allSearchTerms = allSearchTerms.filter(term => term.length >= this.minMatchingChars);
        }
        const regExps = map(allSearchTerms, (term) => new RegExp(term, 'gi'));

        if (this.tagsSearchField) {
          const tags = data[this.tagsSearchField] || [];
          const matched = tags.filter(tag => some(regExps, re => tag.match(re) !== null));
          return map(matched, (tag) => {
            let tagText = tag;
            for (const re of regExps) {
              if (tagText.match(re)) {
                tagText = tagText.replace(re, `<span class='${this.highlightClass}'>$&</span>`)
              }
            }
            return tagText;
          });
        }
        else {
          return [];
        }
      }

    },

    escapedQuery() {
      // replace multiple spaces with a single space
      const escaped = escapeRegExp(sanitize(this.query));
      if (escaped) {
        return escaped.replace(/\s+/g, ' ').trim();
      }
      return escaped;
    },

    actionableItems() {
      return reject(this.matchedItems, (matchedItem) => {
        return this.isDisabledItem(matchedItem)
      })
    },

    matchedItems() {
      if (!this.showOnFocus && (isEmpty(this.query) || this.query.length < this.minMatchingChars)) {
        return []
      }

      let allSearchTerms = this.showAllResults ? [''] : this.escapedQuery.split(this.searchStringToken);

      // terms should have the min matching chars
      allSearchTerms = allSearchTerms.filter(term => term.length >= this.minMatchingChars);

      if (isEmpty(allSearchTerms)) {
        return [];
      }

      // create a regexp for the whole term; this should take priority
      const wholeSearchTerm = this.showAllResults ? '' : this.escapedQuery;
      const wholeRegExp = new RegExp(wholeSearchTerm, 'gi');

      // create a big OR Regexp with all the terms
      const termsRegExp = new RegExp(allSearchTerms.join('|'), 'gi');

      // apply Regexps to all terms so we can filter and sort
      const matchedData = map(this.data, (item) => {
        const searchText = this.searchTermField ? item.data[this.searchTermField] || '' : i.text;
        item.matchLength = 0;
        item.matchType = 0;

        // first search by the whole term, if no match do each term
        const wholeTermMatch = searchText.match(wholeRegExp);
        if (wholeTermMatch && !isEmpty(wholeTermMatch)) {
          item.matchLength = wholeTermMatch[0].length;
          item.matchType = 1;
          item.data.htmlLabel = this.highlightText(item.text);

          if (item.data.description) {
            item.data.htmlDesc = this.highlightText(item.data.description, true);
          }
        }
        else {
          // match on each term instead
          const matchedTerms = searchText.match(termsRegExp);
          if (!isEmpty(matchedTerms)) {
            item.matchLength += reduce(matchedTerms, (sum, accum) => {
              return sum + accum.length;
            }, 0);
            item.matchType = 2;

            item.data.htmlLabel = item.text.replace(termsRegExp, `<span class='${this.highlightClass}'>$&</span>`);

            if (item.data.description) {
              item.data.htmlDesc = item.data.description.replace(termsRegExp, `<span class='${this.highlightClass}'>$&</span>`);
            }
          }
        }

        // try to match on tags; if any set the match type
        if (this.tagsSearchField) {
          const regExps = map(allSearchTerms, (term) => new RegExp(term, 'gi'));
          const tags = item.data[this.tagsSearchField] || [];
          const matchedTags = tags.filter(tag => some(regExps, re => tag.match(re) !== null));
          if (!isEmpty(matchedTags)) {
            item.matchLength += matchedTags.length;
            item.matchType = 3;
          }
        }

        return item;
      });

      return matchedData
        .filter((i) => {
          return i.matchType > 0;
        }).sort((a, b) => {
          if (this.disableSort) {
            return 0;
          }

          // sort by match length desc
          let diff = b.matchLength - a.matchLength;

          // secondary by type asc
          if (diff === 0) {
            diff = a.matchType - b.matchType;
          }

          // tertiary by alpha asc
          if (diff === 0) {
            diff = a.data.id.localeCompare(b.data.id);
          }

          return diff;
        }).slice(0, this.maxMatches)
    }
  },

  methods: {
    highlightText(text, returnNonNullForNoMatch) {
      if (!text) {
        return null;
      }
      const cleanText = sanitize(text)
      if (this.query.length === 0 && returnNonNullForNoMatch) {
        return cleanText
      }

      const re = new RegExp(this.escapedQuery, 'gi')
      if (!returnNonNullForNoMatch && !cleanText.match(re)) {
        return null;
      }

      return cleanText.replace(re, `<span class='${this.highlightClass}'>$&</span>`);
    },

    handleParentInputKeyup(e) {
      switch (e.keyCode) {
        case 40: // down arrow
          this.selectNextListItem()
          break
        case 38: // up arrow
          this.selectPreviousListItem()
          break
        case 13: // enter
          this.hitActiveListItem()
          break
      }
    },

    handleHit(item, evt) {
      this.$emit('hit', item)
      evt.preventDefault()
    },
    hitActiveListItem() {
      if (this.activeListItem < 0) {
        this.selectNextListItem();
      }
      if (this.activeListItem >= 0) {
        this.$emit('hit', this.matchedItems[this.activeListItem])
      }
    },
    isDisabledItem(item) {
      return includes(this.disabledValues, item.text)
    },

    isListItemActive(id) {
      return this.activeListItem === id
    },
    resetActiveListItem() {
      this.activeListItem = -1
    },

    findIndexForNextActiveItem(itemsToSearch, currentSelectedItem) {
      if (!itemsToSearch) {
        itemsToSearch = this.matchedItems
      }
      if (currentSelectedItem === undefined) {
        currentSelectedItem = this.activeListItem
      }

      let nextActiveIndex = findIndex(
        itemsToSearch,
        function(o) { return !this.isDisabledItem(o) }.bind(this),
        currentSelectedItem + 1
      )

      if (nextActiveIndex === BEFORE_LIST_INDEX) {
        nextActiveIndex = findIndex(
          itemsToSearch,
          function(o) { return !this.isDisabledItem(o) }.bind(this)
        )
      }

      return nextActiveIndex
    },

    selectNextListItem() {
      if (this.actionableItems.length <= 0) {
        this.activeListItem = BEFORE_LIST_INDEX
        return true
      }

      this.activeListItem = this.findIndexForNextActiveItem()
    },

    selectPreviousListItem() {
      if (this.actionableItems.length <= 0) {
        this.activeListItem = BEFORE_LIST_INDEX
        return true
      } else if (this.activeListItem === 0) {
        this.activeListItem = BEFORE_LIST_INDEX
      }

      let reversedList = reverse(clone(this.matchedItems))
      let currerntReversedIndex = ((this.matchedItems.length - 1) - this.activeListItem)
      let nextReverseIndex = this.findIndexForNextActiveItem(reversedList, currerntReversedIndex)

      this.activeListItem = (this.matchedItems.length - 1) - nextReverseIndex
    }
  },
  watch: {
    activeListItem(newValue, oldValue) {
      if (!this.$parent.autoClose && this.$parent.isFocused === false) {
        this.$parent.isFocused = true
      }
      if (newValue >= 0) {
        const scrollContainer = this.$refs.suggestionList
        const listItem = scrollContainer.children[this.activeListItem]
        const scrollContainerlHeight = scrollContainer.clientHeight
        const listItemHeight = listItem.clientHeight
        const visibleItems = Math.floor(
          scrollContainerlHeight / (listItemHeight + 20)
        )
        if (newValue >= visibleItems) {
          scrollContainer.scrollTop = listItemHeight * this.activeListItem
        } else {
          scrollContainer.scrollTop = 0
        }
        listItem.focus()
      }
    }
  }
}
</script>
