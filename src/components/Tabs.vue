<template>
  <div>
    <div class="tabs">
      <button v-for="tab in tabs" :key="tab" @click="selectedTab = tab" :class="{ active: selectedTab === tab }">
        {{ tab }}
      </button>
    </div>
    <div class="tab-content">
      <slot :name="selectedTab"></slot>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      selectedTab: 'Tab1',
      tabs: ['Tab1', 'Tab2', 'Tab3']
    };
  }
};
</script>

<style scoped>
.tabs {
  display: flex;
  cursor: pointer;
}
.tabs button {
  padding: 10px;
  margin-right: 5px;
  border: none;
  background-color: #f1f1f1;
}
.tabs button.active {
  background-color: #ccc;
}
.tab-content {
  margin-top: 20px;
}
</style>
