========================
Customising Form display
========================

The :doc:`react_components` provided by newforms can help you get started
quickly and offer a degreee of customisation, but you can completely customise
the way a form is presented by rendering it yourself.

To assist with rendering, we introduce another concept which ties together
Widgets, Fields and Forms:

.. _ref-custom-display-boundfield:

BoundField
==========

BoundField
   A :js:class:`BoundField` is a helper for rendering HTML content for -- and
   related to -- a single Field.

   It ties together the Field itself, the fields's configured Widget, the name
   the field is given by the Form, and the raw user input data and validation
   errors held by a Form.

   BoundFields provide properties and functions for using these together to
   render the different components required to display a field -- its label,
   form inputs and validation error messages -- as well as exposing the
   constituent parts of each of these should you wish to fully customise every
   aspect of form display.

Forms provide a number of methods for creating BoundFields. These are:

* ``form.boundFieldsObj()`` -- returns an object whose properties are the form's
  field names,  with BoundFields as values.
* ``form.boundFields()`` -- returns a list of BoundFields in their form-defined
  order.
* ``form.boundField(fieldName)`` -- returns the BoundField for the named field.

Every object which can generate ``ReactElement`` objects in newforms has a
default ``render()`` method -- for BoundFields, the default ``render()`` for a
non-hidden field calls ``asWidget()``, which renders the Widget the field
is configured with.

A selection of the properties and methods of a BoundField which are useful for
custom field rendering are listed below. For complete details, see the
:doc:`boundfield_api`.

Useful BoundField properties
============================

``bf.name``
   The name of the field in the form.

``bf.htmlName``
   The name the field will be represented by when rendered. If each Form and
   FormSet being used to render user inputs has a unique
   :ref:`prefix <ref-form-prefixes>`, this is guaranteed to be a unique name.

   As such, it's a good candidate if you need a unique ``key`` prop for a React
   component related to each field.

``bf.label``
   The label text for the field, e.g. ``'Email address'``.

``bf.helpText``
   Any help text that has been associated with the field.

``bf.field``
   The :js:class:`Field` instance from the form, that this :js:class:`BoundField`
   wraps. You can use it to access field properties directly.

   Newforms also adds a :ref:`custom property <ref-fields-field-custom>` to the
   Field API -- you can pass this argument when creating a field to store any
   additional, custom metadata you want to associate with the field for later
   use.

Useful BoundField methods
=========================

``bf.errors()``
   Gets an object which holds any validation error messages for the field and
   has a default rendering to a ``<ul class="errorlist">``.

``bf.errorMessage()``
   Gets the first validation error message for the field as a String, or
   ``undefined`` if there are none, making it convenient for conditional display
   of error messages.

``bf.idForLabel()``
   Generates the id that will be used for this field. You may want to use this
   in lieu of ``labelTag()`` if you are constructing the label manually.

``bf.labelTag()``
   Generates a ``<label>`` containing the field's label text, with the
   appropriate ``htmlFor`` property.

``bf.helpTextTag()``
   By default, generates a ``<span className="helpText">`` containing the
   field's help text if it has some configured, but this can be configured with
   arguments.

   .. versionadded:: 0.10

``bf.status()``
   Gets the current validation status of the field as a string, one of:

   * ``'pending'`` -- has a pending async validation.
   * ``'error'`` -- has validation errors.
   * ``'valid'`` -- has neither of the above and data present in
     ``form.cleanedData``.
   * ``'default'`` -- none of the above (likely hasn't been interacted with or
     validated yet).

   .. versionadded:: 0.10

``bf.value()``
   Gets the value which will be displayed in the field's user input.

``boundFields()`` example
=========================

Using these, let's customise rendering of our ContactForm. Rendering things in
React is just a case of creating ``ReactElement`` objects, so the full power of
JavaScript and custom React components are available to you.

For example, let's customise rendering to add a CSS class to our form field rows
and to put the checkbox for the ``ccMyself`` field inside its ``<label>``:

.. code-block:: javascript

   function renderField(bf) {
     var className = 'form-field'
     if (bf.field instanceof forms.BooleanField) {
       return <div className={className}>
         <label>{bf.render()} {bf.label}</label>
         {bf.helpTextTag()} {bf.errors().render()}
       </div>
     }
     else {
       return <div className={className}>
         {bf.labelTag()} {bf.render()}
         {bf.helpTextTag()} {bf.errors().render()}
       </div>
     }
   }

We still don't need to do much work in our component's ``render()`` method:

.. code-block:: javascript

   render: function() {
     return <form action="/contact" method="POST">
       {this.state.form.boundFields.map(renderField)}
       <div>
         <input type="submit" value="Submit"/>{' '}
         <input type="button" value="Cancel" onClick={this.onCancel}/>
       </div>
     </form>
   }

Its initial rendered output is now:

.. code-block:: html

   <form action="/contact" method="POST">
     <div class="form-field"><label for="id_subject">Subject:</label> <input maxlength="100" type="text" name="subject" id="id_subject"></div>
     <div class="form-field"><label for="id_message">Message:</label> <input type="text" name="message" id="id_message"></div>
     <div class="form-field"><label for="id_sender">Sender:</label> <input type="email" name="sender" id="id_sender"></div>
     <div class="form-field"><label for="id_ccMyself"><input type="checkbox" name="ccMyself" id="id_ccMyself"> Cc myself</label></div>
     <div><input type="submit" value="Submit"> <input type="button" value="Cancel"></div>
   </form>

``boundFieldsObj()`` example
============================

The following Form and FormSet will be used to take input for a number of items
to be cooked:

.. code-block:: javascript

   var ItemForm = forms.Form.extend({
     name: Forms.CharField(),
     time: Forms.IntegerField(),
     tend: Forms.ChoiceField({required: false, choices: ['', 'Flip', 'Rotate']})
   })

   var ItemFormSet = forms.formsetFactory(ItemForm, {extra: 3})

The list of item forms will be presented as a ``<table>`` for alignment and
compactness. We could use ``boundFields()`` as above and loop over each form's
fields, creating a ``<td>`` for each one, but what if we wanted to display a
unit label alongside the "time" field and dynamically display some extra content
alongside the "tend" field?

If every field needs to be rendered slightly differently, or needs to be placed
individually into an existing layout, ``boundFieldsObj()`` provides a convenient
way to access the form's BoundFields by field name:

.. code-block:: javascript

   <tbody>
     {itemFormset.forms().map(function(itemForm, index) {
       var fields = itemForm.boundFieldsObj()
       return <tr>
         <td>{fields.name.render()}</td>
         <td>{fields.time.render()} mins</td>
         <td>
           {fields.tend.render()}
           {fields.tend.value() && ' halfway'}
         </td>
       </tr>
     })}
   </tbody>
