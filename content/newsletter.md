+++
title = "Newsletter"
date = "2025-06-20"
description = "Subscribe to get the latest posts and thoughts on impact and engineering"
showDate = false
showReadingTime = false
+++

# Newsletter

Stay updated with my latest thoughts

<style>
.newsletter-form {
  max-width: 500px;
  margin: 2rem 0;
}

.newsletter-form label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
  color: var(--content);
}

.newsletter-form input[type="email"] {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  font-size: 1rem;
  background: var(--entry);
  color: var(--content);
  margin-bottom: 1rem;
  box-sizing: border-box;
}

.newsletter-form input[type="email"]:focus {
  outline: none;
  border-color: var(--tertiary);
}

/* Fix autofill styling for both light and dark themes */
.newsletter-form input[type="email"]:-webkit-autofill,
.newsletter-form input[type="email"]:-webkit-autofill:hover,
.newsletter-form input[type="email"]:-webkit-autofill:focus {
  -webkit-box-shadow: 0 0 0 1000px var(--entry) inset !important;
  -webkit-text-fill-color: var(--content) !important;
  border: 1px solid var(--border) !important;
}

/* Firefox autofill */
.newsletter-form input[type="email"]:-moz-autofill {
  background-color: var(--entry) !important;
  color: var(--content) !important;
}

.newsletter-form input[type="submit"] {
  background-color: var(--primary);
  color: var(--theme);
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: var(--radius);
  cursor: pointer;
  font-size: 1rem;
  transition: opacity 0.2s;
}

.newsletter-form input[type="submit"]:hover {
  opacity: 0.8;
}

.newsletter-form .powered-by {
  margin-top: 1rem;
  color: var(--secondary);
  font-size: 0.9rem;
}

.newsletter-form .powered-by a {
  color: var(--secondary);
  text-decoration: none;
}
</style>

<form
  action="https://buttondown.com/api/emails/embed-subscribe/kaplan"
  method="post"
  target="popupwindow"
  onsubmit="window.open('https://buttondown.com/kaplan', 'popupwindow')"
  class="newsletter-form"
>
  <label for="bd-email">Enter your email</label>
  <input 
    type="email" 
    name="email" 
    id="bd-email" 
    required
  />
  
  <input type="submit" value="Subscribe" />
  
  <p class="powered-by">
    <a href="https://buttondown.com/refer/kaplan" target="_blank">Powered by Buttondown</a>
  </p>
</form>