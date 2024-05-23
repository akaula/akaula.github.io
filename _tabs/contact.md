---
# the default layout is 'page'
icon: fa fa-phone-square
title: Contact
order: 6
---

<!-- … -->
<style>
    form {
    max-width: 600px;
    margin: 0 auto;
    }

    label {
    display: block;
    margin: 0.5em 0 0.2em;
    }

    input, textarea {
    width: 100%;
    padding: 0.5em;
    margin-bottom: 1em;
    }

    button {
    padding: 0.7em 1.5em;
    background-color: #007BFF;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    }

    button:hover {
    background-color: #0056b3;
    }
</style>
<!-- … -->

<form action="https://formspree.io/f/xoqgzlyb" method="POST">
  <label for="full-name">
    Full Name:
    <input type="text" name="name" id="full-name" placeholder="First and Last">
  </label>
  <label>
    Your email:
    <input type="email" name="email">
  </label>
  <label>
    How can we help you?:
    <textarea name="message"></textarea>
  </label>
  <!-- your other form fields go here -->
  <button type="submit">Send</button>
</form>
