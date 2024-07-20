<template>
    <div class='card'>
      {{ rank }} of {{ suit }}
    </div>
  </template>
  
  <script setup lang='ts'>
  import { defineProps } from 'vue'
  
  interface CardProps {
    suit: string
    rank: string
  }
  
  defineProps<CardProps>()
  </script>
  
  <style scoped>
  .card {
    border: 1px solid black;
    padding: 10px;
    margin: 5px;
    display: inline-block;
  }
  </style>
  