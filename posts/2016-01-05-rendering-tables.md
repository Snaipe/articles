---
layout: post
categories: [Python, Algorithm]
tags: [
  python,
  algorithm,
  pretty-print,
  table,
  tabular,
  ascii,
  proof-of-concept,
  programming,
  article,
  coding,
  code,
]
title: "[Python] A simple algorithm for drawing complex tables"
share: true
comments: true
---

I was surprised to see, during the development of [rst2ansi][rst2ansi], that there were no
simple python modules to pretty-print complex tables with ascii characters.

[rst2ansi]: https://github.com/Snaipe/python-rst2ansi

Most of them only supported simple tables with uni-line cells, and God forbid you want
to print cells that span multiple columns or rows.

I went on and made a little algorithm to help me draw these tables.

## The algorithm

We want to draw an arbitrary table, for instance:

    +-----+-----+-----+-----+
    |  A  |  B  |  C  |  D  |
    +-----+-----+-----+-----+
    |  E  |        F        |
    +-----+-----+-----------+
    |  G  |     |           |
    +-----+  I  |     J     |
    |  H  |     |           |
    +=====+=====+===========+

<style>
  table.algo-step {
     border-spacing: 0;
     border-collapse: separate;
  }

  table.graph-table tbody tr td {
    width: 60px;
    border: 1px solid black;
    padding: 0px;
    text-align: center;
  }

  table.graph-table tbody tr {
    height: 60px;
  }

  table.algo-step tbody {
    border: 1px dashed black;
  }

  table.algo-step tbody tr td {
    width: 50px;
    border: 1px dashed black;
    padding: 0px;
    text-align: center;
    border-top: 0;
    border-left: 0;
  }

  table.algo-step tbody tr {
    height: 50px;
  }

  table.algo-step tbody tr.ch-up td {
    border-top: 2px solid black;
  }

  table.algo-step tbody tr td.ch-left {
    border-left: 2px solid black;
  }

  table.table-highlight tbody tr.ch-up td {
    border-top: 2px solid orange;
  }

  table.table-highlight tbody tr td.ch-left {
    border-left: 2px solid orange;
  }

  table.algo-step .cell-highlight {
    border-bottom: 2px solid orange;
    border-right: 2px solid orange;
    color: orange;
  }

  table.algo-step .cell-normal {
    border-bottom: 2px solid black;
    border-right: 2px solid black;
    color: black;
  }

  table.graph-table .blue {
    background-color: #9fa8da;
    color: white;
  }

  table.graph-table .purple {
    background-color: #ab47bc;
    color: white;
  }

  table.graph-table .red {
    background-color: #ef5350;
    color: white;
  }

  table.graph-table .green {
    background-color: #388e3c;
    color: white;
  }

  .twocol {
    -webkit-column-count: 2;
    -moz-column-count: 2;
    column-count: 2;
  }

  .twocol > table {
    -webkit-column-span: 1;
    -moz-column-span: 1;
    column-span: 1;
  }

  .threecol {
    -webkit-column-count: 3;
    -moz-column-count: 3;
    column-count: 3;
  }
</style>

The first thing we need to define is what exactly constitute a row
in this table, and a traversal order for each cell:

<div class='twocol'>
<table class='graph-table'>
  <tbody>
    <tr class='blue'>
      <td>Row 1</td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr class='green'>
      <td>Row 2</td>
      <td colspan="3"></td>
    </tr>
    <tr class='purple'>
      <td>Row 3</td>
      <td rowspan="2"></td>
      <td rowspan="2" colspan="2"></td>
    </tr>
    <tr class='red'>
      <td>Row 4</td>
    </tr>
  </tbody>
</table>

<table class='graph-table'>
  <tbody>
    <tr>
      <td>1</td>
      <td>2</td>
      <td>3</td>
      <td>4</td>
    </tr>
    <tr>
      <td>5</td>
      <td colspan="3">6</td>
    </tr>
    <tr>
      <td>7</td>
      <td rowspan="2">8</td>
      <td rowspan="2" colspan="2">9</td>
    </tr>
    <tr>
      <td>10</td>
    </tr>
  </tbody>
</table>
</div><br>

Once done, we can straight-forwardly draw the table by iterating over
the cells using the above order:

1. Draw the top and left table borders
2. For each cell draw its bottom and right borders.

Here is a step-by-step visualisation of the process for the above table:

<div class="threecol">
  <table class="algo-step table-highlight">
    <caption>Step 1</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left"></td>
        <td></td>
        <td></td>
        <td></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td colspan="3"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 4</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-highlight">C</td>
        <td></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td colspan="3"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 7</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-normal">D</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">E</td>
        <td class="cell-highlight" colspan="3">F</td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 10</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-normal">D</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">E</td>
        <td class="cell-normal" colspan="3">F</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">G</td>
        <td class="cell-normal" rowspan="2">I</td>
        <td class="cell-highlight" rowspan="2" colspan="2">J</td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 2</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-highlight">A</td>
        <td></td>
        <td></td>
        <td></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td colspan="3"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 5</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-highlight">D</td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td colspan="3"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 8</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-normal">D</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">E</td>
        <td class="cell-normal" colspan="3">F</td>
      </tr>
      <tr>
        <td class="ch-left cell-highlight">G</td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 11</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-normal">D</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">E</td>
        <td class="cell-normal" colspan="3">F</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">G</td>
        <td class="cell-normal" rowspan="2">I</td>
        <td class="cell-normal" rowspan="2" colspan="2">J</td>
      </tr>
      <tr>
        <td class="ch-left cell-highlight">H</td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 3</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-highlight">B</td>
        <td></td>
        <td></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td colspan="3"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 6</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-normal">D</td>
      </tr>
      <tr>
        <td class="ch-left cell-highlight">E</td>
        <td colspan="3"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
        <td rowspan="2"></td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 9</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-normal">D</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">E</td>
        <td class="cell-normal" colspan="3">F</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">G</td>
        <td class="cell-highlight" rowspan="2">I</td>
        <td rowspan="2" colspan="2"></td>
      </tr>
      <tr>
        <td class="ch-left"></td>
      </tr>
    </tbody>
  </table><br>

  <table class="algo-step">
    <caption>Step 12</caption>
    <tbody>
      <tr class="ch-up">
        <td class="ch-left cell-normal">A</td>
        <td class="cell-normal">B</td>
        <td class="cell-normal">C</td>
        <td class="cell-normal">D</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">E</td>
        <td class="cell-normal" colspan="3">F</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">G</td>
        <td class="cell-normal" rowspan="2">I</td>
        <td class="cell-normal" rowspan="2" colspan="2">J</td>
      </tr>
      <tr>
        <td class="ch-left cell-normal">H</td>
      </tr>
    </tbody>
  </table><br>
</div><br>

(Note that this algorithm could be used to draw tables on any medium, not just for ascii pretty-printing.)

It is immediately apparent that for this algorithm to work, it needs precise information
on the table:

* It needs, of course, the number of rows and columns and the contents of the cells,
* It needs the layout of each cell, i.e. which cells spans multiple rows and/or columns.
* It needs the height of each row and the width of each column.

This means that we have to define a proper data structure to represent these informations.

### The implementation

## Setting up the stage

The first thing we have to define is the way to represent our data. Given a table like this:

    +------------------------+------------+----------+----------+
    | Header row, column 1   | Header 2   | Header 3 | Header 4 |
    | (header rows optional) |            |          |          |
    +========================+============+==========+==========+
    | body row 1, column 1   | column 2   | column 3 | column 4 |
    +------------------------+------------+----------+----------+
    | body row 2             | Cells may span columns.          |
    +------------------------+------------+---------------------+
    | body row 3             | Cells may  | Cells may span both |
    +------------------------+ span rows. | rows and columns.   |
    | body row 4             |            |                     |
    +========================+============+=====================+

What would be the best way to represent it?

One could think at first to use multi-dimensional list, but we get hit in the face
when trying to deal with cells spaning multiple columns and rows.

It turns out the best format to represent a table with, albeit a little (and by that I mean "lot")
verbose, is to use a tree:

{% highlight python linenos %}
TABLE = {
    'node': 'table',
    'children': [
      {
        'node': 'head',
        'children': [
            {
              'node': 'row',
              'children': [
                {'node': 'cell', 'data': 'Header row, column 1\n(header rows optional)'},
                {'node': 'cell', 'data': 'Header 2'},
                {'node': 'cell', 'data': 'Header 3'},
                {'node': 'cell', 'data': 'Header 4'},
              ]
            }
          ]
      },
      {
        'node': 'body',
        'children': [
          {
            'node': 'row',
            'children': [
              {'node': 'cell', 'data': 'body row 1, column 1'},
              {'node': 'cell', 'data': 'column 2'},
              {'node': 'cell', 'data': 'column 3'},
              {'node': 'cell', 'data': 'column 4'},
            ]
          },
          {
            'node': 'row',
            'children': [
              {'node': 'cell', 'data': 'body row 2'},
              {'node': 'cell', 'data': 'Cells may span columns.', 'morecols': 2},
            ],
          },
          {
            'node': 'row',
            'children': [
              {'node': 'cell', 'data': 'body row 3'},
              {'node': 'cell', 'data': 'Cells may span rows.', 'morerows': 1},
              {'node': 'cell', 'data': 'Cells may span both rows and columns.', 'morerows': 1, 'morecols': 1},
            ],
          },
          {
            'node': 'row',
            'children': [
              {'node': 'cell', 'data': 'body row 4'},
            ],
          }
        ]
      }
    ]
  }
{% endhighlight %}

This doesn't really look easy to make, but it matters not: we will only use
this format as an intermediate structure, meant to be generated from another
source.

Looks familiar? It just so happens that this structure looks a little bit like
the one HTML uses:

{% highlight html linenos %}
<table>
  <thead>
    <tr>
      <td>Header row, column 1<br>(header rows optional)</td>
      <td>Header 2</td>
      <td>Header 3</td>
      <td>Header 4</td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>body row 1, column 1</td>
      <td>column 2</td>
      <td>column 3/td>
      <td>column 4/td>
    </tr>
    <tr>
      <td>body row 2</td>
      <td colspan="3">Cells may span columns.</td>
    </tr>
    <tr>
      <td>body row 3</td>
      <td rowspan="2">Cells may span rows.</td>
      <td rowspan="2" colspan="2">Cells may span both rows and columns.</td>
    </tr>
    <tr>
      <td>body row 4</td>
    </tr>
  </tbody>
</table>
{% endhighlight %}

We could make a function to transform these HTML tables into our intermediate format,
for instance.

### Sizing the table up

The other data we need for this algorithm to work is the dimensions of each row and columns.

The implementation of this function depends on the use case and the strategy you want.
Some possible ideas:

* Find the width of a column:
  * Use a fixed width
  * For each column, use the mean of widths of all cell contents in this specific column
  * Use the length of the column label
* Find the height of a row:
  * Use the maximum number of lines the contents of each cell in a row can fill.

### Implementing the algorithm

The major component of the algorithm is the iteration over the cells. Since we
defined a tree-like data structure, a good iteration technique is walking over
each node with a visitor.

We start by defining a `Visitor` class that implements `visit`.
`visit` is a method that calls `'visit_' + node_name` on the node,
visits all of its children, then calls `'depart_' + node_name`.

We also define a `SkipChildren` exception, which, if raised by a `visit_`
function, will tell the visitor not to visit the children of the current node.

{% highlight python linenos %}
class SkipChildren(Exception):
  pass

class Visitor(object):

  def visit(self, node):
    """
    Visit a generic node. Calls 'visit_' + node_name on the node,
    then visit its children, then calls 'depart_' + node_name.
    """
    def noop(node):
      pass

    try:
      visit_fn = getattr(self, 'visit_' + node['node'])
    except AttributeError:
      visit_fn = noop

    try:
      depart_fn = getattr(self, 'depart_' + node['node'])
    except AttributeError:
      depart_fn = noop

    try:
      visit_fn(node)
      for n in node.get('children', []):
        self.visit(n)
    except SkipChildren:
      return
    finally:
      depart_fn(node)
{% endhighlight %}

With our visitor class, we can finally define our table printer:

{% highlight python linenos %}
# Note: for the sake of this article's brevity, the code below
# is actually python pseudocode. A working implementation can
# be found in the git repository linked at the end of this article.

class TablePrinter(Visitor):

  def __init__(self):
    self.row = 0
    self.col = 0

  def visit_table(self, node):
    _draw_initial_table(node['colspec'], node['rowspec'])

  def visit_row(self, node):
    self.col = 0

  def depart_row(self, node):
    self.row += 1

  def visit_cell(self, node):
    # Get position & dimensions

    pos = self.row, self.col
    dim = self._get_cell_dimensions(node, pos)

    spanned_cols, spanned_rows, width, height = dim

    # Draw the cell

    self._draw_bottom_border(dim, pos, node)
    self._draw_right_border(dim, pos, node)

    self._draw_cell_contents(dim, pos, node)

    # Update the position & stop recursing

    self.col += spanned_cols
    raise SkipChildren

  def _get_cell_dimensions(self, node, pos):
    """
    Returns the number of columns and rows this cell spans,
    and its width in character columns and height in lines.

    Args:
        node: the node of the cell.
        pos: the row and column indices of this cell.
    """
    raise NotImplementedError

  def _draw_initial_table(self, widths, heights):
    """
    Draw the leftmost and upmost borders of the table,
    and fills the defined rectangle with spaces.
    Args:
        widths: the widths of each columns
        heights: the heights of each rows
    """
    total_width = sum(widths) + (len(widths) + 1)
    total_height = sum(heights) + (len(heights) + 1)

    raise NotImplementedError

  def _draw_bottom_border(self, dim, pos, cell):
    """
    Draw a bottom border for this cell
    Args:
        dim: the dimensions of the cell
        pos: the position of the cell in the table
        cell: the cell properties
    """
    raise NotImplementedError

  def _draw_right_border(self, dim, pos, cell):
    """
    Draw a bottom border for this cell
    Args:
        dim: the dimensions of the cell
        pos: the position of the cell in the table
        cell: the cell properties
    """
    raise NotImplementedError

  def _draw_right_border(self, dim, pos, cell):
    """
    Draw the contents of this cell
    Args:
        dim: the dimensions of the cell
        pos: the position of the cell in the table
        cell: the cell properties
    """
    raise NotImplementedError

{% endhighlight %}

## Wrapping up

I published on github an ascii table renderer using this algorithm
as a proof of concept [here][table2ascii]. The code isn't perfect,
and needs to be cleaned up a bit, but should be overall understandable.

This algorithm is simple enough for purposes where there are no border styles.
For broader ones, one could either change this algorithm to make it overwrite
previous borders, or use the guidelines as described in the [official w3c document on CSS2 properties of HTML tables][w3c-tables].

[table2ascii]: https://github.com/Snaipe/table2ascii
[w3c-tables]: http://www.w3.org/TR/CSS2/tables.html
