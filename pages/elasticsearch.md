---
layout: page
title: ES全文检索
titlebar: elasticsearch
subtitle: <span class="mega-octicon octicon-graph"></span>&nbsp;&nbsp;
     Elasticsearch 是一个分布式、可扩展、实时的搜索与数据分析引擎
menu: elasticsearch
css: ['blog-page.css']
permalink: /elasticsearch
---

<div class="row">

    <div class="col-md-12">

        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='elasticsearch'  or post.keywords contains 'elasticsearch' %}
                <li class="posts-list-item">
                    <div class="posts-content">
                        <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                        <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
                        <span class='circle'></span>
                    </div>
                </li>
                {% endif %}
            {% endfor %}
        </ul> 

        <!-- Pagination -->
        {% include pagination.html %}

        <!-- Comments -->
       <div class="comment">
         {% include comments.html %}
       </div>
    </div>

</div>
<script>
    $(document).ready(function(){

        // Enable bootstrap tooltip
        $("body").tooltip({ selector: '[data-toggle=tooltip]' });

    });
</script>