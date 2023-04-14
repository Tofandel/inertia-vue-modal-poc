# Inertia Vue Modal

> **Warning**
>
> I stopped development of this package and will not be using it anymore, there is a lot of issues with it and it is definitely not something I'd recommend using in production. Have a look at [momentum-modal](https://github.com/lepikhinb/momentum-modal) for an alternative or [hybridly](https://github.com/hybridly/hybridly) for complete inertia alternative (and much more)
>
> **Warning**

## Limitations

This POC is a very early draft, so these limitations will probably be fixed soon.

* No support for nested modals (modal in modal)
* It doesn't use browser history navigation

## Installation

### Client-side installation

```bash
composer require tofandel/inertia-vue-modal && npm i vendor/tofandel/inertia-vue-modal
```

In your layout, you need to add the `InertiaModal` component, here is an example using the Quasar Dialog

```vue

<template>
  <div class="min-h-screen">
    <nav></nav>

    <!-- Page Content -->
    <main>
      <slot />
    </main>

    <InertiaModal>
      <template #default="{close, props}">
        <QDialog
            :model-value="true"
            v-bind="props/* props contains the modalProps option */"
            @update:model-value="close"
        >
          <ModalSlot />
        </QDialog>
      </template>
    </InertiaModal>
  </div>
</template>

<script setup>
import { InertiaModal, ModalSlot } from "@tofandel/inertia-vue-modal";
</script>
```

##### Axios dependency

It's very important that your project only uses one version of axios because of the use of interceptors, if you use webpack and your packages have conflicting versions of axios they will each be bundled separately and interceptors won't work
To resolve this issue make sure that you add this to your webpack.config.js
```js
const path = require('path');

module.exports = {
  resolve: {
    alias: {
      //...
      'axios': path.resolve('node_modules/axios/dist/axios.js'),
    },
  },
  //...
}
```

### Server-side installation

In your Laravel application, you need to extend our HandlesInertiaModalRequest middleware

```php
<?php

namespace App\Http\Middleware;

//...
use Tofandel\InertiaVueModal\HandlesInertiaModalRequest;

class HandleInertiaRequests extends HandlesInertiaModalRequest
{
//...
}
```

## Usage

Since we added the `InertiaModal` component, the global `$inertia` object now has a `visitInModal` method. This allows you to make an Inertia visit that loads into the modal.

You can use this method, for example, in the `@click` handler of a button:

```vue
<button @click="$inertia.visitInModal('/user/create')">Load in modal</button>
```

Instead of using the method in your template, you can also use it in your script:

```vue

<script setup>
import { Inertia } from "@inertiajs/inertia";
Inertia.visitInModal('/user/create', {
  // Visit options
  
  // This is passed as additional page props to the page component of your modal
  pageProps: {
    componentProps1: 'page',
  },
  // This is passed as `props` in the InertiaModal default template
  modalProps: {
    modalProps1: 'modal',
  }
});
</script>
```

### Update the page you want to load into a modal

In most cases, the `/user/create` endpoint renders a form that's wrapped into a template, maybe with other components and components to style the form. Here's a simple example of what the `UserCreate.vue` component might look like:

```vue
<template>
  <QLayout>
    <QPageContainer>
      <form @submit.prevent="form.post('/user/store')">
        <input type="text" v-model="form.name">
        <input type="email" v-model="form.email">

        <button type="submit">Login</button>
      </form>
    <QPageContainer>
  </QLayout>
</template>


<script setup>
import { useForm } from '@inertiajs/inertia-vue3';

const form = useForm({
  name: "",
  email: "",
});
</script>
```

To load this form into a modal, we don't want the sidebar, footer, and styling from the `form-panel` component. We want just the `form` itself!

To accomplish this, you need to do three things:

1. Add the `IsModalable` mixin to your component.
2. Wrap your *whole* component into the `Modalable` component.
3. Move the `form` to a separate `#toModal` template and replace it with a `ToModal` component.

```vue

<template>
  <!-- the new Modalable root component -->
  <ModalableWrapper>
    <!-- the 'old' root component -->
    <!-- This will be wrapped around the #modal template but only when NOT within a modal -->
    <QLayout>
      <QPageContainer>
        <!-- the previous location of the form, replaced by the ModalSlot component, which will render the #modal template -->
        <ModalSlot />
      </QPageContainer>
    </QLayout>

    <template #modal-only>
      <!-- This will be wrapped around the #modal template but only when within a modal -->
      <QCard>
        <QCardSection>
          <!-- will render the #modal template -->
          <ModalSlot />
        </QCardSection>
      </QCard>
    </template>
    <template #modal>
      <!-- This is the content that is always shown -->
      <!-- the 'new' location of the form -->
      <form @submit.prevent="form.post('/user.store')">
        <input type="text" v-model="form.name">
        <input type="email" v-model="form.email">

        <button type="submit">Login</button>
      </form>
    </template>
  </ModalableWrapper>
</template>

<script setup>
import { ModalableWrapper, ModalSlot } from "@tofandel/inertia-vue-modal";
import { useForm } from '@inertiajs/inertia-vue3';

// This is where pageProps is received
defineProps({
  componentProps1: String,
  otherRenderProps: null,
});

const form = useForm({
  name: "",
  email: "",
});
</script>
```

Now when you visit `/user/create`, nothing has changed! You still have your layout and `form-panel` styling. But when you load this component into a modal, it will only render the `form`.

### Handling redirects

By default, redirects are handled as any other Inertia request. For example you're visiting `/user`, you open `/user/create` in a modal, and after a successful submit, you redirect the user to the detail page of the newly created user:

```php
public function store(UserStoreRequest $request)
{
    $user = User::create(...);

    return redirect()->route('user.show', $user);
}
```

You might not always want to route to the detail page. Luckily, you don't have to update your server-side implementation.

The `visitInModal` method accepts a second argument that can either a Boolean or a callback. Instead of redirecting the user, the user stays on the same page, and you can manually handle the event with the callback. This callback is executed after a successful request, for example, when the new user is stored in the database.

```javascript
this.$inertia.visitInModal('/user/create', {
  redirectBack: (evt) => {
    // Do something
  },
});
```
