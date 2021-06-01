<template>
  <div id="app" class="container mx-auto pb-12">
    <h1 class="font-bold text-4xl text-blue-700 mt-12">Enchantment helper</h1>
    <p v-if="loadState.loading">Loading...</p>
    <p v-else-if="loadState.error">Failed to get data</p>
    <div v-else>
      <Popup
        ref="popup"
        @finish="(e) => addItem(e.data.itemId, e.choices, e.data.enchantments)"
      />
      <Heading>Enchantable items</Heading>
      <hr />
      <div class="pt-5 block">
        <Item
          v-for="item of enchantableItems"
          :key="item.id"
          :item="item"
          @click="chooseItem(item)"
        />
      </div>
      <Heading>Chosen items</Heading>
      <hr />
      <div class="pt-5 grid gap-2 grid-cols-1 md:grid-cols-3">
        <div v-for="(item, i) of chosenItems" :key="i" class="bg-gray-100 p-5">
          <h2 class="font-bold text-xl hover:text-red-600 cursor-pointer px-2" @click="removeItem(i)">{{getItem(item.itemId).displayName}}</h2>
          <div v-for="enchant of item.enchantments" :key="enchant" class="block mb-2 p-2 border-1 rounded">
            {{ getEnchant(enchant).displayName }} {{ getEnchant(enchant).maxLevel }}
          </div>
        </div>
      </div>
      <Heading>Required enchantments</Heading>
      <hr />
      <div class="pt-5">
        <div v-for="enchant of enchantList" :key="enchant.name">
          <div>{{ enchant.count }}x {{ enchant.enchant.displayName }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Heading from "./components/heading.vue";
import Item from "./components/item.vue";
import Popup from "./components/popup.vue";

function arrayEqual(arr1, arr2) {
  // Check if the arrays are the same length
  if (arr1.length !== arr2.length) return false;

  // Check if all items exist and are in the same order
  for (var i = 0; i < arr1.length; i++) {
    if (arr1[i] !== arr2[i]) return false;
  }

  // Otherwise, return true
  return true;
}

export default {
  components: {
    Heading,
    Item,
    Popup,
  },
  data() {
    return {
      loadState: {
        error: false,
        loading: false,
        success: false,
      },
      enchantmentPath: "/enchantments.json",
      itemPath: "/items.json",
      enchantments: [],
      items: [],
      chosenItems: [],
    };
  },
  async mounted() {
    this.getEnchantmentData();
  },
  computed: {
    enchantableItems() {
      return this.items.filter(
        (v) => v.enchantCategories && v.enchantCategories.length > 0
      );
    },
    enchantList() {
      const list = [];
      for (let item of this.chosenItems) {
        for (let enchant of item.enchantments) {
          const found = list.find((v) => v.name === enchant);
          if (found) {
            found.count++;
            continue;
          }
          list.push({
            count: 1,
            name: enchant,
          });
        }
      }
      return list.map((v) => ({
        count: v.count,
        enchant: this.getEnchant(v.name),
      })).sort((a,b) => a.count < b.count);
    },
  },
  methods: {
    async getEnchantmentData() {
      this.loadState.error = false;
      this.loadState.loading = true;
      try {
        const enchantmentData = await fetch(this.enchantmentPath).then((v) =>
          v.json()
        );
        const itemData = await fetch(this.itemPath).then((v) => v.json());
        this.$set(this, "enchantments", enchantmentData);
        this.$set(this, "items", itemData);
        this.loadState.loading = false;
        this.loadState.success = true;
      } catch (err) {
        console.error(err);
        this.loadState.error = true;
        this.loadState.loading = false;
        this.loadState.success = false;
        return;
      }
    },
    getItem(id) {
      return this.items.find((v) => v.id === id);
    },
    getEnchant(name) {
      console.log(name);
      return this.enchantments.find((v) => v.name === name);
    },
    getEnchantmentCategory(category) {
      return this.enchantments.filter((v) => v.category === category);
    },
    chooseItem(item) {
      const data = this.anyliseEnchantments(item);
      if (data.choices.length == 0) {
        this.addItem(item.id, [], data.enchantments);
        return;
      }

      this.$refs.popup.create(data.choices, {
        itemId: item.id,
        enchantments: data.enchantments,
      });
    },
    addItem(item, choices, enchantments) {
      choices = choices.flatMap((v) => v);
      choices.push(...enchantments);
      this.chosenItems.push({
        enchantments: choices,
        itemId: item,
      });
    },
    removeItem(index) {
      this.chosenItems = this.chosenItems.filter((_, i) => i !== index);
    },
    anyliseEnchantments(item) {
      // get all enchantments
      let enchantments = [];
      for (let category of item.enchantCategories) {
        enchantments.push(...this.getEnchantmentCategory(category));
      }
      enchantments = enchantments.filter(
        (v, i, a) => a.findIndex((t) => t.id === v.id) === i
      );

      // search for mutually exclusive enchantments
      const normalEnchantments = [];
      let conflictingEnchantments = [];
      for (let enchant of enchantments) {
        if (enchant.curse) continue;
        const conflicting = enchantments
          .filter((v) => enchant.exclude.includes(v.name))
          .map((v) => v.name);
        if (conflicting.length > 0) {
          conflictingEnchantments.push(enchant);
          continue;
        }
        normalEnchantments.push(enchant.name);
      }

      // process conflicting enchantments
      conflictingEnchantments = conflictingEnchantments.map((v) => ({
        name: v.name,
        values: [v.name],
        exclude: [...v.exclude, v.name].sort(),
        excludeWithoutSelf: [...v.exclude].sort(),
      }));

      // turn into choices
      let choices = [];
      while (conflictingEnchantments.length > 0) {
        const conflict = conflictingEnchantments[0];
        conflictingEnchantments = conflictingEnchantments.filter(
          (v) => !arrayEqual(v.exclude, conflict.exclude)
        );
        choices.push(conflict);
      }

      // combine choices
      let hadChange = true;
      while (hadChange) {
        hadChange = false;
        for (let conflict of choices) {
          const sameConflict = choices.find(
            (v) =>
              v.name !== conflict.name &&
              arrayEqual(conflict.excludeWithoutSelf, v.excludeWithoutSelf)
          );
          if (sameConflict) {
            conflict.values = [...conflict.values, sameConflict.name];
            choices = choices.filter((v) => v.name !== sameConflict.name);
            hadChange = true;
            break;
          }
        }
      }

      // combine more choices
      let choicePages = [];
      for (let choice of choices) {
        const links = choicePages.find((v) => v.depends.includes(choice.name));
        if (links) {
          links.choices.push(choice);
          links.depends.push(...choice.excludeWithoutSelf);
          continue;
        }
        choicePages.push({
          choices: [choice],
          depends: [...choice.exclude],
        });
      }

      // transform choices into pages
      choicePages = choicePages.map((v) => {
        if (v.choices.length == 1) {
          return {
            type: "poll",
            choices: v.choices[0].exclude.map((v) =>({ value: v, display: this.getEnchant(v).displayName })),
          };
        }
        return {
          type: "poll",
          choices: v.choices.map((v) => v.values.map(v=>({ value: v, display: this.getEnchant(v).displayName }))),
        };
      });

      // return data
      return {
        enchantments: normalEnchantments,
        choices: choicePages,
      };
    },
  },
};
</script>
