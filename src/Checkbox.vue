<template>
<label>
    <input type="checkbox">
    {{ label}}
</label>


</template>



<script setup>

const props = defineProps({
    label: String
})

defineEmits(['check', 'uncheck'])


</script>