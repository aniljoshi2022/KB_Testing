### Sticky Header

Sticky headers are useful for providing users with a clear understanding of their current position on the webpage. Here is an example of how you can achieve this using both CSS and JavaScript.

#### CSS Implementation

```css
css .sticky-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  background-color: #333;
  padding: 10px;
}

.sticky-header .logo {
  display: inline-block;
  vertical-align: middle;
}

.sticky-header nav ul {
  list-style-type: none;
  margin: 0;
  padding: 0;
  overflow: hidden;
}

.sticky-header nav li {
  float: left;
}

.sticky-header a {
  color: #fff;
  text-decoration: none;
  display: block;
  padding: 10px;
}

.sticky-header a:hover {
  background-color: #444;
}

body {
  margin: 0;
}
```