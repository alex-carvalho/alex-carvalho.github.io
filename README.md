# Alex Carvalho Blog 🚀

This is the repository for my personal study blog, built with **Jekyll** and hosted on **GitHub Pages**.

## How to Run the Blog Locally

To run and test the blog on your local machine before pushing commits to GitHub, follow the steps below:

### Prerequisites
* **Ruby (version >= 3.0):** Using a version manager like `rbenv`.
* **Bundler:** The Ruby package manager.

---

### Step-by-Step Guide

1. **Install project dependencies:**
   Open the terminal in the root folder of the repository and run the command below to install Jekyll and its associated plugins:
   ```bash
   bundle install
   ```

2. **Start the local Jekyll server:**
   Start the development server with the command:
   ```bash
   bundle exec jekyll serve
   ```

3. **Access in the Browser:**
   Open your browser and navigate to the address provided in the terminal:
   👉 **[http://localhost:4000](http://localhost:4000)** (or `http://127.0.0.1:4000`)

---

## Project Structure

* `_layouts/`: Contains the main HTML templates (`default.html`, `home.html`, `post.html`).
* `_posts/`: Your blog posts written in Markdown format.
* `assets/`:
  * `css/style.scss`: Where all the custom styling and responsive design live.
  * `images/`: Contains the featured images for the posts.
* `index.md`: The home page that manages the post flow.
* `archive.md`: The page displaying the list of posts by tags.
