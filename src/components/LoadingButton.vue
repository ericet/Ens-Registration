<template>
  <div>
    <div v-if="!isLoading && !committed && !registered">
      <button type="button" @click="commit" class="btn btn-primary">
        Commit
      </button>
    </div>
    <div v-else-if="isLoading && !committed && !registered">
      <button class="btn btn-secondary" type="button" disabled>
        <span
          class="spinner-grow spinner-grow-sm"
          role="status"
          aria-hidden="true"
        ></span>
        Please Wait...
      </button>
    </div>
    <div v-else-if="!isLoading && committed && !registered">
      <button type="button" @click="register()" class="btn btn-success">
        Register
      </button>
    </div>
    <div v-else class="alert alert-success" role="alert">
      Congraluations! You've succesfully registered <b>{{ name }}.eth</b>!
    </div>
  </div>
</template>

<script>
export default {
  name: 'LoadingButton',
  props: ['isLoading', 'committed', 'registered', 'commit', 'register', 'name'],
};
</script>
