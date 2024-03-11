---
layout: post
show_meta: true
title: Using useMemo hook
header: Using useMemo hook
date: 2024-03-11 00:00:00
summary: Using useMemo hook to control form submission
categories: reactjs 
author: Chee Yeo
---

[useMemo Hook]: https://react.dev/reference/react/useMemo
[Skip re-rendering of component]: https://react.dev/reference/react/useMemo#skipping-re-rendering-of-components

In a ReactJS app, I have a form as a parent component that needs to pass its values as query parameters to a child component, which uses these query parameters to make an API call to update its view.

The issue I encountered is whenever the form values changes, the state of the configuration object shared between the 2 components change. This is detected by the child component via its **useEffect** hook, causing multiple requests to be made to the external api whenever a form value changes.

The ideal solution is to only update the child component if a form submission is made, by [Skip re-rendering of component].

We can make use of the [useMemo Hook] hook which takes a component and a shared state value, which would cache the rendering of the component until the shared state value changes.

For example, we could have a form component which passes its values to a child component to render its view.

{% highlight javascript %}
import React, { useState, useEffect, useMemo } from 'react';


const FormX = () => {
   const [formValues, setFormValues] = useState({
    per_page: 10,
    query: "language:python",
    order: "desc",
    sort: "stars"
  });

  const [isSubmitted, setIsSubmitted] = useState(false);

  const callAPI = (data) => {
    // Calls the api...
  };

  const handleChange = (event) => {
      const { name, value } = event.target;
      setFormValues({ ...formValues, [name]: value });
  };

  const handleSubmit = (e) => {
      e.preventDefault();
      setIsSubmitted(true);
  };


  // Prevent from re-rendering on every form change
  const child = useMemo(() => <View data={formValues} onSubmit={setIsSubmitted} fetchRepos={callAPI} />, [isSubmitted]);


  return (
    <div>
        <form id="formu" onSubmit={handleSubmit} className="row gy-2 gx-3 align-items-center">
          ....
        </form>

        {child}
    </div>
  )
}


const View = ({ data, onSubmit, fetchRepos }) => {
    useEffect(() => {
      // Pass the setIsSubmit handler here to change state to cause the view to re-render
      onSubmit(false);

      fetchRepos(data);
  }, [data, onSubmit]);

  return (
    <!-- HTML to render view -->
  )
}

{% endhighlight %}

From the example above, we have **FormX** as the parent component which contains a child component of **View** that renders the response from an API call with the form values from its parent, which are passed to it as query parameters.*

We invoke `useMemo` with the child **View** component. The **View** component takes a data parameter; a callback which references the form submission state ( setIsSubmitted ); and a callback that invokes the API. We also set `isSubmitted` as the state to watch. If it changes, the child view `useEffect` will be called re-rendering the component.

The initial value of `isSubmitted` is set to false. Within the `handleChange` callback, the `isSubmitted` value remains false so while the form updates, it doesn't trigger any updates on the child component.

The `handleSubmit` callback gets invoked after the form is submitted. This sets the `isSubmitted` value to be true via `setIsSubmitted(true)`. This invalidates the `useMemo` cache as its set to watch `isSubmitted` state, which triggers `useEffect` in the child component. We pass in `setIsSubmitted` as `OnSubmit` parameter. which is set to false in `useEffect`. This causes the child component to re-render, invoking `fetchRepos`.

Hope it helps.

H4ppy H4ck1ng!