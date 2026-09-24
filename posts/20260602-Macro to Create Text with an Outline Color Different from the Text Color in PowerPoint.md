---
title: "Macro to Create Text with an Outline Color Different from the Text Color in PowerPoint"
description: "Create outlined text in PowerPoint with a VBA macro that duplicates a selected text box, derives the outline color from its background, and groups the layers."
pubDatetime: 2026-06-02T12:57:43.176Z
---

In PowerPoint, you may want to add an outline in a different color while preserving the original text color. If you use the standard text outline feature, the outline can extend inward and make the fill color appear compressed or partially obscured.

This macro avoids that problem by overlaying two text boxes: **one for the outline** and **one for the text fill**.

## Purpose of This Macro

The goal of this macro is to make it easy to create text in PowerPoint that:

* Keeps the original text fill color intact
* Adds a thick outline in a different color around the text
* Prevents the outline from crushing or narrowing the visible text fill

This is especially useful when placing text over photos or dark-colored shapes where readability is important.

## How It Works

The macro duplicates the selected text box.

The original text box receives the outline, while the duplicated text box is placed on top and serves as the text fill layer. Finally, the two text boxes are grouped together.

With a standard text outline, the inside of the text can sometimes appear narrower or the fill color may become less visible. By stacking two text boxes, the appearance remains much cleaner.

## How to Use It

### 1. Create a Text Box

First, create a text box in PowerPoint.

Enter the text you want to display.

### 2. Set a Background Color on the Text Box

Next, assign a background color to the text box.

This background color will be used as the outline color. For example, if you want a white outline, set the text box background color to white.

### 3. Select Only One Text Box

Before running the macro, select only the target text box.

If multiple objects or non-text shapes are selected, the macro will not run.

### 4. Run the Macro

With the text box selected, execute the macro.

The macro retrieves the text box background color and applies it as a 6-point text outline. It then creates a duplicate text box in the same position, overlays it, and groups the two text boxes together.

## Macro Code

```vb
Sub TextBoxOutlineFromBackground()

    Dim shpA As Shape
    Dim shpC As Shape
    Dim fillColor As Long
    Dim sr As ShapeRange
    Dim sld As Slide

    ' Get the currently selected object
    If ActiveWindow.Selection.Type <> ppSelectionShapes Then
        MsgBox "Please select a shape.", vbExclamation
        Exit Sub
    End If

    If ActiveWindow.Selection.ShapeRange.Count <> 1 Then
        MsgBox "Please select only one object.", vbExclamation
        Exit Sub
    End If

    Set shpA = ActiveWindow.Selection.ShapeRange(1)

    ' Verify that object A is a text box
    If shpA.Type <> msoTextBox Then
        MsgBox "The selected object is not a text box.", vbExclamation
        Exit Sub
    End If

    If Not shpA.HasTextFrame Then
        MsgBox "The selected object does not contain a text frame.", vbExclamation
        Exit Sub
    End If

    If Not shpA.TextFrame.HasText Then
        MsgBox "The selected text box contains no text.", vbExclamation
        Exit Sub
    End If

    ' Verify that object A has a background color
    If shpA.Fill.Visible <> msoTrue Then
        MsgBox "The selected text box does not have a background color.", vbExclamation
        Exit Sub
    End If

    ' Use the background color as color B
    fillColor = shpA.Fill.ForeColor.RGB

    shpA.Fill.Visible = msoFalse

    ' Duplicate and create C
    Set shpC = shpA.Duplicate(1)

    shpC.Left = shpA.Left
    shpC.Top = shpA.Top
    shpC.Width = shpA.Width
    shpC.Height = shpA.Height

    ' Remove A's background fill
    shpA.Fill.Visible = msoFalse

    ' Apply a 6pt outline to A using the background color
    With shpA.TextFrame2.TextRange.Font.Line
        .Visible = msoTrue
        .ForeColor.RGB = fillColor
        .Weight = 6
        .Transparency = 0
    End With

    ' Group A and C on the same slide
    Set sld = shpA.Parent
    Set sr = sld.Shapes.Range(Array(shpA.Name, shpC.Name))
    sr.Group.Select

End Sub
```

## Add It to the Quick Access Toolbar or Ribbon

This macro becomes much more convenient if you add it to the Quick Access Toolbar or the Ribbon.

If you use it frequently, assigning it to a button is faster than opening the Macro dialog each time.

The following article explains how to add macros to the Quick Access Toolbar or Ribbon:

[https://qiita.com/FollowUser/items/25604896eb78c8d3a053](https://qiita.com/FollowUser/items/25604896eb78c8d3a053)

## Notes

To use this macro, you must first assign a background color to the text box. Without a background color, the macro cannot determine which color to use for the outline and will not run.

Also, only a single text box can be selected when running the macro. If multiple objects are selected, or if a regular shape is selected instead of a text box, an error message will be displayed.

## Summary

This macro allows you to create PowerPoint text with a different-colored outline while preserving the original text fill color.

By overlaying one text box for the outline and another for the text fill, the interior of the text remains clear and readable. This technique is particularly useful when placing text over photographs or when creating highly visible slide titles and headings.
