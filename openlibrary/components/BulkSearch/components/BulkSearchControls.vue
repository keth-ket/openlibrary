<template>
  <div class="bulk-search-controls">
    <div>
      <p v-if="showColumnHint">
        Please include a header row. Supported columns include: "Title", "Author".
      </p>

      <select
        v-model="selectedValue"
        class="sampleBar"
      >
        <option
          v-for="sample in sampleData"
          :key="sample.source"
          :value="sample.text"
        >
          {{ sample.name }}
        </option>
      </select>
      <textarea
        v-model="bulkSearchState.inputText"
        placeholder="Enter your books..."
      />
      <br>
      <div class="progressCarousel">
        <div class="progressCard">
          <div class="numeral">
            1
          </div>
          <div class="info">
            <div class="heading">
              <h3> Extract Books</h3>
              <p><i>How to convert your books above into structured information, like title and author.</i></p>
            </div>
            <label><strong> Extractor</strong> <br> <select v-model="bulkSearchState._activeExtractorIndex">
              <option
                v-for="extractor, index in bulkSearchState.extractors"
                :key="index"
                :value="index"
              >
                {{ extractor.label }}
              </option>
            </select></label>

            <label v-if="showApiKey"><strong>OpenAI API Key</strong> <br>
              <input
                v-model="bulkSearchState.extractionOptions.openaiApiKey"
                class="api-key-bar"
                :type="showPassword ? 'password' : 'text'"
                placeholder="OpenAI API key here...."
                @focus="showPassword = false"
                @blur="showPassword = true"
              >
            </label>


            <button
              :disabled="loadingExtractedBooks"
              @click="extractBooks"
            >
              {{ extractBooksText }}
            </button>
          </div>
        </div>
        <div
          class="progressCard"
          :class="{ progressCardDisabled: matchBooksDisabled}"
        >
          <div class="numeral">
            2
          </div>
          <div class="info">
            <div class="heading">
              <h3>Match Books</h3>
              <p><i>Once structured data has been found, it's time to match it to a book in OpenLibrary!</i></p>
            </div>
            <label><strong>Options</strong> <br>
              <input
                v-model="bulkSearchState.matchOptions.includeAuthor"
                type="checkbox"
              > Use author in
              search query
            </label>
            <button
              :disabled="loadingMatchedBooks || matchBooksDisabled"
              @click="matchBooks"
            >
              {{ matchBooksText }}
            </button>
          </div>
        </div>
        <div
          class="progressCard"
          :class="{ progressCardDisabled: createListDisabled }"
        >
          <div class="numeral">
            3
          </div>
          <div class="info">
            <div class="heading">
              <h3 class="heading">
                Save your Matches
              </h3>
              <p class="heading">
                <i> Now that you've found your books, why not save them to your reading log? Or a list?</i>
              </p>
            </div>
            <div v-if="bulkSearchState?.matchedBooks.length <50">
              <a
                id="listMakerLink"
                :href="bulkSearchState.listUrl"
                target="_blank"
              ><button :disabled="createListDisabled">Add to list</button></a>
            </div>
            <form
              v-else
              method="POST"
              action="/account/lists/add"
              target="_blank"
            >
              <input
                type="hidden"
                name="seeds"
                :value="bulkSearchState?.listString"
              >
              <button type="submit">
                Add to List
              </button>
            </form>
          </div>
        </div>
      </div>
    </div>
    <div v-if="bulkSearchState.errorMessage">
      <p
        v-for="error in bulkSearchState.errorMessage"
        :key="error"
      >
        {{ error }}
      </p>
    </div>
  </div>
</template>

<script>
import {sampleData} from '../utils/samples.js';
import { BulkSearchState} from '../utils/classes.js';
import { buildSearchUrl } from '../utils/searchUtils.js'
export default {

    props: {
        bulkSearchState: BulkSearchState
    },
    data() {
        return {
            selectedValue: '',
            showPassword: true,
            sampleData: sampleData,
            loadingExtractedBooks: false,
            loadingMatchedBooks: false,
            matchBooksDisabled: true,
            createListDisabled: true,
        }
    },
    computed: {
        showApiKey(){
            if (this.bulkSearchState.activeExtractor) return 'model' in this.bulkSearchState.activeExtractor
            return false
        },
        extractBooksText(){
            if (this.loadingExtractedBooks) return 'Loading...'
            return 'Extract Books'
        },
        matchBooksText(){
            if (this.loadingMatchedBooks) return 'Loading...'
            return 'Match Books'
        },
        showColumnHint(){
            if (this.bulkSearchState.activeExtractor) return this.bulkSearchState.activeExtractor.name === 'table_extractor'
            return false
        },
    },
    watch: {
        selectedValue(newValue) {
            if (newValue!==''){
                this.bulkSearchState.inputText = newValue;
            }
        }
    },
    methods: {
        togglePasswordVisibility(){
            this.showPassword= !this.showPassword
        },
        async extractBooks() {
            this.loadingExtractedBooks = true
            const extractedData = await this.bulkSearchState.activeExtractor.run(this.bulkSearchState.extractionOptions, this.bulkSearchState.inputText)
            this.bulkSearchState.matchedBooks = extractedData
            this.loadingExtractedBooks = false
            this.matchBooksDisabled = false;
            this.createListDisabled = true;
        },
        async matchBooks() {
            const fetchSolrBook = async function (book, matchOptions) {
                try {
                    const data = await fetch(buildSearchUrl(book, matchOptions, true))
                    return await data.json()
                }
                catch (error) {}
            }
            this.loadingMatchedBooks = true
            for (const bookMatch of this.bulkSearchState.matchedBooks) {
                bookMatch.solrDocs = await fetchSolrBook(bookMatch.extractedBook, this.bulkSearchState.matchOptions)
            }
            this.loadingMatchedBooks = false
            this.createListDisabled = false
        },

    }
}
</script>

<style lang="less">

.bulk-search-controls{
    padding:20px;
}
label input {
    flex: 1;
}

.sampleBar{
    float:right;
    margin-bottom: 5px;
}
textarea {
    width: 100%;
    height: 120px;
    display: flex;
    resize: vertical;
    box-sizing: border-box;
}
.progressCarousel{
    display:flex;
    overflow-x:scroll;
    column-gap:10px;
}
.progressCard{
    background-color:#C7E3FC;
    padding: 16px;
    width: min(450px, 66vw);
    height:fit-content;
    border-radius:30px;
    display:flex;
    column-gap:16px;
    flex-shrink:0;
    .info{
        display:flex;
        flex-direction:column;
        row-gap:10px;
        .heading{
            color:#0376B8;
            h3{
                margin:0px;
            }
            p{
                margin:0px;
            }
        }
        select{
            width: 100%;
        }
        button{
            background-color:#0376B8;
            color:white;
            border-radius:4px;
            box-shadow: none;
            border:none;
            padding: 0.5rem;
            transition:  background-color 0.2s;
            min-width:140px;
            align-self:center;
            &:not([disabled]) {
                cursor:pointer;
                &:hover{
                    background-color:#014c78;
                }
            }
        }
    }
    .numeral{
        border-radius: 50%;
        height:48px;
        width:48px;
        background-color:white;
        color:#0376B8;
        font-weight:bold;
        justify-content:center;
        flex-shrink: 0;
        display:flex;
        align-items:center;
        font-size:24px;
    }
}

.progressCardDisabled{
    opacity:50%;
}


.api-key-bar{
    width:100%;
    box-sizing:border-box;
}
</style>
