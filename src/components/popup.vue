<template>
  <div
    v-if="isOpen"
    class="
      fixed
      top-0
      left-0
      w-screen
      h-screen
      bg-black bg-opacity-75
      flex
      items-center
      justify-center
    "
  >
    <div
      class="bg-white max-w-full p-12 inline-block rounded"
      style="width: 32rem"
    >
      <div v-if="current.type === 'poll'">
        <h1 class="font-bold text-2xl mb-2">Choose enchantment</h1>
        <hr class="mb-5" />
        <p
          v-for="(choice, j) of current.choices"
          :key="j"
          class="
            bg-gray-200
            hover:bg-indigo-700
            hover:text-white
            px-5
            py-1
            rounded
            mb-2
            cursor-pointer
          "
          @click="setChoice(currentPage, j)"
        >
          {{ choiceToString(choice) }}
        </p>
      </div>
      <div v-else>
        <p>Unknown page type</p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      isOpen: false,
      currentPage: 0,
      pages: [],
      choices: [],
      data: {},
    };
  },
  computed: {
    current() {
      return this.pages[this.currentPage];
    },
  },
  methods: {
    create(pages, data) {
      this.data = data;
      this.pages = pages;
      this.currentPage = 0;
      this.choices = [];
      this.open();
    },
    open() {
      this.isOpen = true;
    },
    close() {
      this.isOpen = false;
    },
    setChoice(page, choice) {
      this.choices.push(this.pages[page].choices[choice].value);
      this.currentPage++;
      if (this.currentPage >= this.pages.length) this.finish();
    },
    finish() {
      this.close();
      this.$emit("finish", {
        choices: this.choices,
        data: this.data,
      });
    },
    choiceToString(c) {
      if (c.constructor === Array && c.length == 1)
        c = c[0];
      if (c.constructor === Array) {
        c = [...c];
        const last = c.pop();
        return `${c.map(v=>v.display).join(", ")} & ${last.display}`
      }
      return `${c.display}`;
    }
  },
};
</script>
