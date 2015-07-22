---
layout: post
title: The Church of Xcode - Simple Emacs Keybindings
feature-img: "img/emacs.jpg"
---

Long has the holy war between emacs and vi been waged. I've used both, and I don't
really care either way. However, OS X text fields everywhere come with secret
support for limited (and sometimes modified) Emacs keybindings. So, regardless
of your position in the great text editor wars you might as well learn a few keybindings
because _you can use them anywhere you see a text field_. Learning a
few basic keybindings will empower you as an iOS/Mac developer and as an OS X user.

## Emacs Modifier Keys

Whenever you look at emacs shortcuts you'll see something like this:

    C-v
    M-v

A little translation is needed: C is the control key (⌃) and M is the "META" key,
which in our case is the alt/option key (⌥). Since we're not really a member of the
church of emacs, from now on we'll refer to our modifier keys using ⌃ and ⌥.

## Remapping Caps Lock

The ⌃ key is not in the easiest position to stretch your fingers to, and it gets used a lot
as a key modifier. So much so, you might want to consider remapping a less useful key that's in a good
position. A common choice for this is the Caps Lock key. If you're like me you don't do a lot of
shouting when you type, and when you do you can hold down the shift key for a minute. So, it's a prime
choice.

Simply go to your Keyboard preferences, hit the modifier keys button in the bottom right,
and change the Caps Lock dropdown to "⌃ Control".

![Remapping Caps Lock](/img/remapping-caps-lock.png "Remapping Caps Lock")

Remapping your keyboard is really easy, and your friends will think you're hardcore.

## Cursor Movement

The point of keybindings is to keep your fingers on the keyboard where they belong; you
want to avoid as much as possible moving your hand to mouse. Even using the arrow keys
requires you to move your fingers away from their home row, and should be avoided where
possible.

| ⌃F | Move Cursor **F**orward |
| ⌃B | Move Cursor **B**ackward |
| ⌃P | Move Cursor U**p** |
| ⌃N | Move Cursor Dow**n** |
| ⌃E | Move to **E**nd of Line (also ⌘→) |
| ⌃A | Move to Beginning of Line ... (**A** is the first letter of the alphabet?) (also ⌘←) |

If you hold down shift with any of these, you will move the selection as you move your cursor.

**You can also use ⌃P and ⌃N to move the selection in your autocomplete results up and down!**

Nobody likes to move the cursor one character at a time when you have a long ways to go. In Emacs,
if you use the meta key instead of control, you can move forward and back by entire words.
If you try to do this in Mac ⌥F and ⌥B instead insert the fancy characters ƒ and ∫, respectively.
This is a little tricky because Mac also allows users to use the ⌥ key to insert special characters.

We can add our own keybindings to override this behavior in Xcode by adding out own Key Bindings in Xcode
preferences (which I do suggest), but part of our goal here is to learn key bindings that work across the system. These key bindings are available system wide, although in modified form—you must hold down both
the control and meta modifier keys.

| ⌥⌃F | Move Cursor **F**orward a word (also ⌥→) |
| ⌥⌃B | Move Cursor **B**ackward a word (also ⌥←) |

Once again, holding shift while using either of these will move your selection. (Although it
requires holding down 3 modifier keys! You'll get lots of finger dexterity).

## Text Deletion

Sometimes you want to delete text. These keybindings will help you delete efficiently.

* Deleting by _word_ deletes whole words delimited by whitespace or other non-word characters.
* Deleting by _subword_ will delete individual words in camelCasedWords (camel, Cased, and Words are each in subwords)

| ⌃⌫ | Backward delete a subword |
| ⌃⌦ | Forward delete a subword |
| ⌥⌫ | Backward delete a word |
| ⌥⌦ | Forward delete a word |
| ⌃k | Delete to end of line (**k**ill) (also  ⌘⌦) |
| ⌘⌫ | Delete to beginning of line |

(⌫ is the backspace key, and ⌦ is the Delete key)

## Exercise

You can look over these shortcuts till the cows come home, but unless you practice them you will
never realize their utility.

One exercise I like to do is to _try not to use the arrow keys at all_. Now, sometimes I find using the arrow
keys to be more efficient (like moving forward/back by word or in the cases of ⌘↑ and ⌘↓ to move to the top and bottom of the file),
but forcing myself to not use them helps me practice my other keybindings and I realize which operations
I can actually do more quickly without the arrow keys.

---

## Conclusion

Emacs of course has many, many more keybindings. With Xcode's Key Binding
preferences you can try to imitate more of them, but these are just some standard OS-wide
key bindings that will help you text edit more efficiently.
