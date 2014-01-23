# Grid system

## @media (min-width : @screen-??-min)

```css

// Small grid
//
// Columns, offsets, pushes, and pulls for the small device range, from phones
// to tablets.

@media (min-width: @screen-sm-min) {
  .make-grid-columns-float(sm);
  .make-grid(@grid-columns, sm, width);
  .make-grid(@grid-columns, sm, pull);
  .make-grid(@grid-columns, sm, push);
  .make-grid(@grid-columns, sm, offset);
}


// Medium grid
//
// Columns, offsets, pushes, and pulls for the desktop device range.

@media (min-width: @screen-md-min) {
  .make-grid-columns-float(md);
  .make-grid(@grid-columns, md, width);
  .make-grid(@grid-columns, md, pull);
  .make-grid(@grid-columns, md, push);
  .make-grid(@grid-columns, md, offset);
}


// Large grid
//
// Columns, offsets, pushes, and pulls for the large desktop device range.

@media (min-width: @screen-lg-min) {
  .make-grid-columns-float(lg);
  .make-grid(@grid-columns, lg, width);
  .make-grid(@grid-columns, lg, pull);
  .make-grid(@grid-columns, lg, push);
  .make-grid(@grid-columns, lg, offset);
}

```

1. The _md_, _sm_, _lg_ in above codes are used as a string to match the switch like code.

*****

## .clearfix

```css

// Clearfix
// Source: http://nicolasgallagher.com/micro-clearfix-hack/
//
// For modern browsers
// 1. The space content is one way to avoid an Opera bug when the
//    contenteditable attribute is included anywhere else in the document.
//    Otherwise it causes space to appear at the top and bottom of elements
//    that are clearfixed.
// 2. The use of `table` rather than `block` is only necessary if using
//    `:before` to contain the top-margins of child elements.
.clearfix() {
  &:before,
  &:after {
    content: " "; // 1
    display: table; // 2
  }
  &:after {
    clear: both;
  }
}

```

1. This is a good way to do a clearfix.
2. _&_ actually is a little like _this_ , it will involve the selecotr/class invoved it.

*****

## .make-row

```css

// Creates a wrapper for a series of columns
.make-row(@gutter: @grid-gutter-width) {
  margin-left:  (@gutter / -2);
  margin-right: (@gutter / -2);
  &:extend(.clearfix all);
}
```

1. If you want to include all the nested elements of the extended style, the _all_ should be add to the selector in the extend expression.
2. By default, the nested elements of the extended style will not be included.

*****

## .make-grid-columns-float

```css
.make-grid-columns-float(@class) {
  .col(@index) when (@index = 1) { // initial
    @item: ~".col-@{class}-@{index}";
    .col((@index + 1), @item);
  }
  .col(@index, @list) when (@index =< @grid-columns) { // general
    @item: ~".col-@{class}-@{index}";
    .col((@index + 1), ~"@{list}, @{item}");
  }
  .col(@index, @list) when (@index > @grid-columns) { // terminal
    @{list} {
      float: left;
    }
  }
  .col(1); // kickstart it
}
```

1. Use _~_ to be a prefix to set this string directly to variant, without less compile.
2. To join and process this kind of selector string by _when_.
3. Only _true_ to be true for _when_, the other values like 1, 100, "1" will be false.

*****

## percentage function

```css
.calc-grid(@index, @class, @type) when (@type = width) and (@index > 0) {
  .col-@{class}-@{index} {
    width: percentage((@index / @grid-columns));
  }
}
.calc-grid(@index, @class, @type) when (@type = push) {
  .col-@{class}-push-@{index} {
    left: percentage((@index / @grid-columns));
  }
}
.calc-grid(@index, @class, @type) when (@type = pull) {
  .col-@{class}-pull-@{index} {
    right: percentage((@index / @grid-columns));
  }
}
.calc-grid(@index, @class, @type) when (@type = offset) {
  .col-@{class}-offset-@{index} {
    margin-left: percentage((@index / @grid-columns));
  }
}
```

1. it will converts a floating point into a percentage string.
2. it will look like below

```css
.col-xs-12 {
  width: 100%;
}
.col-xs-11 {
  width: 91.66666666666666%;
}
.col-xs-10 {
  width: 83.33333333333334%;
}
```
*****


## .make-grid

```css
// Basic looping in LESS
.make-grid(@index, @class, @type) when (@index >= 0) {
  .calc-grid(@index, @class, @type);
  // next iteration
  .make-grid((@index - 1), @class, @type);
}

```

1. we can use above _when_ codes to do a iteration.

