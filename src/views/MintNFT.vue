<template>
  <div class="page-wrapper">
    <h1 class="home-page-title">{{ appTitle }}</h1>
    Mint

    <div>
      <b-form inline>
        <label class="sr-only" for="inline-form-input-name">Name</label>
        <b-form-input
          id="inline-form-input-name"
          class="mb-2 mr-sm-2 mb-sm-0"
          placeholder="Jane Doe"
          ></b-form-input>

          <label class="sr-only" for="inline-form-input-username">Username</label>
          <b-input-group prepend="@" class="mb-2 mr-sm-2 mb-sm-0">
            <b-form-input id="inline-form-input-username" placeholder="Username"></b-form-input>
          </b-input-group>

          <b-form-checkbox class="mb-2 mr-sm-2 mb-sm-0">Remember me</b-form-checkbox>

          <b-button variant="primary">Save</b-button>
      </b-form>
    </div>

  </div>
</template>

<script>
import { mapState } from 'vuex'

export default {
  head() {
    return {
      title: {
        inner: 'Mint ergo-names NFT',
      },
      meta: [
        {
          name: 'description',
          content: `${this.appTitle} home page`,
          id: 'desc',
        },
      ],
    }
  },
  computed: mapState('app', ['appTitle']),
}
</script>

<style lang="scss" scoped>
@import '@/theme/variables.scss';

.page-wrapper {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  .logo {
    margin-bottom: 3rem;
  }

  .home-page-title {
    text-align: center;
  }

  .documentation-link {
    display: inline-block;
    font-size: 1.2rem;
    color: #fff;
    background-color: #5d6788;
    padding: 0.8rem 1.6rem;
    border-radius: 4px;
    transition: background-color 0.1s ease;
    box-sizing: border-box;
    text-decoration: none;
    width: fit-content;
    font-weight: 500;
  }
}
</style>
