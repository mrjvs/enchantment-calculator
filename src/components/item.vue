<template>
  <div
    class="
      rounded
      bg-gray-200
      p-2
      inline-block
      mr-2
      mb-2
      hover:bg-gray-400
      cursor-pointer
    "
    @click="$emit('click')"
  >
    <p>{{ item.displayName }}</p>
  </div>
</template>

<script>
export default {
  props: {
    item: {
      type: Object,
      default: () => ({}),
    },
  },
};
</script>
