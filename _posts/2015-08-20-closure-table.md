---
layout:     post
show_meta: true
title:      Closure Table pattern for nested hierarchical data
header:     Closure Table pattern for nested hierarchical data
date:       2015-08-20 15:56:00
summary:  Using the Closure Table pattern for nested hierarchical data
categories: rails ruby database
author: Chee Yeo
---

Sometimes in web applications there is a need to store a hierarchical data structure in the database. A good example would be a comment list or thread. Since each comment can accept replies and in turn, the replies themselves can also accept further replies, this leads to a need to maintain a tree data structure to represent the comments when persisted.

Normally, we could just use a __parent_id__ column in the Comments table, such as an Adjacent List, to keep track of its parents and descendants. However this approach presents several problems:

* You can only query up to two levels in the hierarchy since there is a direct relationship between the parent and child.

* You need to add MORE JOIN statements in order to query further into the hierarchy.

* The application would need to fetch all the rows from the database into the application before it can rebuild the hierarchy, slowing the application down by consuming more memory.

* Maintaining referential integrity is difficult as moving or deleting records involve series of steps to accomplish.

An alternative approach would be to use the Closure Table pattern. It stores all the metadata about the hierarchy i.e. parent-child relationship and depth in a separate table, providing a clean separation of concerns from the main data table.

Hence it is possible to store hierarchies of indefinite depth using this approach.

Lets look at the previous example of having a comment list in the context of a Rails application.

We could create a separate table called __comment_hierarchy__, to store the parent child relationship and depth. This would include the following migration:

{% highlight ruby linenos %}
class CreateCommentHierarchies < ActiveRecord::Migration
  def change
    create_table :comment_hierarchies, id: false do |t|
      t.integer :ancestor_id, null: false
      t.integer :descendant_id, null: false
      t.integer :generations, null: false
    end

    add_index :comment_hierarchies, [:ancestor_id, :descendant_id, :generations],
      unique: true,
      name: "comment_anc_desc_idx"

    add_index :comment_hierarchies, [:descendant_id],
      name: "comment_desc_idx"
  end
end
{% endhighlight %}

The __ancestor_id__ column would point to the parent comment; the __descendant_id__ to the child; and __generations__ refer to the depth of the tree.

For instance, a single comment without any children would be represented as:

{% highlight sql linenos %}
+-------------+---------------+-------------+
| ancestor_id | descendant_id | generations |
+-------------+---------------+-------------+
|           1 |             1 |           0 |
+-------------+---------------+-------------+
{% endhighlight %}

The same comment with a single child would be represented as:

{% highlight sql linenos %}
+-------------+---------------+-------------+
| ancestor_id | descendant_id | generations |
+-------------+---------------+-------------+
|           1 |             1 |           0 |
|           1 |             2 |           1 |
|           2 |             2 |           0 |
+-------------+---------------+-------------+
{% endhighlight %}

Here, [2,2,0] represents the new comment #2 and [1,2,1] represents the relationship: parent of comment #2 is comment #1 and it is at a depth of 1 (direct child of comment #1)

To find the descendants of comment #1:

{% highlight sql linenos %}
SELECT c.comment_id from comments as c JOIN comment_hierarchies as t on c.comment_id = t.descendant_id where t.ancestor_id=1;
{% endhighlight %}

To find the ancestors of comment #1:

{% highlight sql linenos %}
SELECT c.comment_id from comments as c JOIN comment_hierarchies as t on c.comment_id = t.descendant_id where t.ancestor_id=1;
{% endhighlight %}

Normally, we want to display the complete comment list on a webpage. This involves fetching the data and reconstructing the tree. We can use a single query to do so:

{% highlight ruby linenos %}
  class Comment < ActiveRecord::Base
  ...
  ...
    def self.hash_tree
      generation_sql = <<-SQL.strip_heredoc
        INNER JOIN(
          SELECT descendant_id, MAX(generations) as depth
          FROM comment_hierarchies
          GROUP BY descendant_id
        ) AS generation_depth
        ON comments.comment_id = generation_depth.descendant_id
      SQL

      tree_scope = Comment.joins(generation_sql).order("generation_depth.depth, created_at DESC")

      self.build_hash_tree(tree_scope)
    end

    ....
  end
{% endhighlight %}

The query above fetches all the descendant_id and its depth from comment_hierarchies table through an inner select, which is then joined with the main comments table through the descendant_id. This data is then used to construct a hash table of comments as keys and its descendants as arrays of values:

{% highlight ruby linenos %}

  class Comment < ActiveRecord::Base
    ...

    def self.build_hash_tree(tree_scope)
      tree = ActiveSupport::OrderedHash.new
      id_to_hash = {}

      tree_scope.each do |ea|
        h = id_to_hash[ea.id] = ActiveSupport::OrderedHash.new
        if ea.root? || tree.empty? # We're at the top of the tree.
          tree[ea] = h
        else
          id_to_hash[ea.parent_id][ea] = h
        end
      end

      tree
    end

    ...
  end
{% endhighlight %}

We can render the comments hash using a recursive helper to check for the presence of nested comments:

{% highlight ruby linenos %}
  module ApplicationHelper
    def comments_tree_for(comments)
      comments.map do |comment, nested_comments|
        render(comment) +
            (nested_comments.size > 0 ? content_tag(:div, comments_tree_for(nested_comments), class: "replies") : nil)
      end.join.html_safe
    end
  end

  # within the view

  <%= comments_tree_for @comments %>

{% endhighlight %}

The only downside to this approach is the need to maintain the comment_hierarchies table through new inserts and deletions. However, compared to other approaches, this is by far the most efficient.

In terms of scalability, reconstructing an entire comments list in memory might be expensive if the depth level if high but a limit or constraint can be set on the initial query to prevent it going beyond a certain depth.

Since there is a lot more going on within the application than can be explained here, I have created a demo rails application which can be found here: [https://github.com/cheeyeo/Closure-Table-Rails4](https://github.com/cheeyeo/Closure-Table-Rails4){:target="_blank"}

In addition, there is a gem called [closure_tree](https://github.com/mceachen/closure_tree){:target="_blank"} which encapsulates the Closure Table pattern and more advanced options. Some of the code shown above is adapted from this gem as I wanted to see how difficult it would be to roll my own solution.

Happy Hacking!.





