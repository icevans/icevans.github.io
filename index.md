This is just a place for me to publish thoughts and lessons as I learn ruby programming through [Launch School](https://launchschool.com/).

        <section class="blog-list">
        {% for post in site.posts %}
          <article class="blog-item">
            <h1><a href="{{ post.url }}">{{ post.title }}</a></h1>
            {{ post.date | date: '%B %d, %Y' }}
            {{ post.excerpt }}
            {% endfor %}
          </article>
        </section>
