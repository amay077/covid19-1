<template>
  <div class="TextCard">
    <h2 v-if="title" class="TextCard-Heading">
      <a v-if="link" :href="link" target="_blank" rel="noopener">
        {{ title }}
      </a>
      <template v-else>
        {{ title }}
      </template>
    </h2>
    <!-- eslint-disable vue/no-v-html -->
    <p v-if="body" v-sanitaize class="TextCard-Body" v-html="body" />
    <!-- eslint-disable vue/no-v-html -->
    <p class="TextCard-Body">
      <slot />
    </p>
  </div>
</template>

<script lang="ts">
import { Vue, Prop, Component } from 'vue-property-decorator'

@Component
export default class TextCard extends Vue {
  @Prop({
    default: '',
    required: false
  })
  title!: string

  @Prop({
    default: '',
    required: false
  })
  link!: string

  @Prop({
    default: '',
    required: false
  })
  body!: string
}
</script>

<style lang="scss">
.TextCard {
  @include card-container();
  padding: 20px;
  margin-bottom: 20px;
  &-Heading {
    @include card-h1();
    margin-bottom: 12px;
    a {
      @include card-h1();
    }
  }
  &-Body {
    * {
      @include body-text();
    }
    a {
      word-break: break-all;
      color: $link;
      text-decoration: none;
    }
  }
}
</style>
