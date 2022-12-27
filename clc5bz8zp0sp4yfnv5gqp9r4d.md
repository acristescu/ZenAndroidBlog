# Using the Jetpack Compose's VisualTransformation to create a credit card text input

![Using the Jetpack Compose's VisualTransformation to create a credit card text input](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091275030/Y8tO26ZkA.png align="left")

In this article we're going to look at how to create a fully-featured text edit field specifically for inputting credit card information. We're going to do this by leveraging the VisualTransformation mechanism of Jetpack Compose's TextField. The repo for this article is available [here](https://github.com/acristescu/ComposeVisualTransformationStudy).

## What we're trying to achieve

We want to create a text field that exhibits the following behaviours:

* initially displays a greyed out mask of 16 digits
    
* as the user types in digits, the mask only shows the missing digit positions
    
* digits are grouped in 4 groups of 4 digits each, separated by two spaces
    
* you should only ever have a maximum of 16 digits. If the user types more at the end, they are ignored. If the user goes back a few positions and types the last digit is truncated
    

![Using the Jetpack Compose's VisualTransformation to create a credit card text input](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091278349/Q2RuGfFEH.gif align="left")

## The layout

Let's start by creating a new activity and setting the following simple layout:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            CardInputTextEditTheme {
                // A surface container using the 'background' color from the theme
                Surface(color = MaterialTheme.colors.background) {
                    Screen()
                }
            }
        }
    }
}

@Composable
fun Screen() {
    var cardNumber by rememberSaveable { mutableStateOf("") }
    Column {
        Row (modifier = Modifier.padding(all = 10.dp)) {
            Text(
                text = "Card number",
                fontSize = 14.sp,
                modifier = Modifier.weight(1f)
            )
            BasicTextField(
                value = cardNumber,
                onValueChange = { cardNumber = it },
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
            )
        }
        Box(modifier = Modifier
            .height(1.dp)
            .padding(start = 10.dp)
            .fillMaxWidth()
            .background(Color.Gray)
        )

        Spacer(modifier = Modifier.height(20.dp))
        Text(text = "Actual value:\n$cardNumber")
    }
}
```

This creates the boilerplate for the above layout, but the TextField is completely vanilla.

## The VisualTransformation

The compose textfield has a new feature that allows the programmer to specify a transformation that is applied to the text inside a textfield before it is shown to the user. This is most useful for example when filling in a password, where we could replace any string the user types with the \* or similar character.

### A simple example

As a warm-up, let's write a simple VisualTransformation that changes whatever the user types into the text "Jetpack Compose is great".

![Using the Jetpack Compose's VisualTransformation to create a credit card text input](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091281575/CHvhI2_nZ.gif align="left")

To achieve this, let's change the TextField to look like this:

```kotlin
            BasicTextField(
                value = cardNumber,
                onValueChange = { cardNumber = it },
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
                visualTransformation = {
                    TransformedText(
                        AnnotatedString("Jetpack Compose is great".repeat(100).take(it.length)),
                        OffsetMapping.Identity
                    )
                }
            )
```

What Compose does is that every time it needs to render the TextField it will call the function we've supplied as a visualTransformation and it will render the result. In our case, we're just repeating "Jetpack Compose is great" 100 times and keeping the first n characters (where n is the number of characters the user has typed). Since we're not changing the length of the string for now, this will suffice and the second parameters is just an OffsetMapping that does nothing.

### A more complicated example

What if we want to change the length of the displayed string? Well, in that case the second parameter of TransformedText comes into effect. That is a function that allows Compose to map any output character to a source character and vice-versa.

Let's make a VisualTransformation that changes the text by adding a dot after every character:

```kotlin
            BasicTextField(
                value = cardNumber,
                onValueChange = { cardNumber = it },
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
                visualTransformation = {
                    TransformedText(
                        AnnotatedString.Builder().run {
                            it.forEach {
                                append(it)
                                append(".")
                            }
                            toAnnotatedString()
                        },
                        object: OffsetMapping {
                            override fun originalToTransformed(offset: Int): Int {
                                return offset * 2
                            }

                            override fun transformedToOriginal(offset: Int): Int {
                                return offset / 2
                            }
                        }
                    )
                }
            )
```

Building the AnnotatedString is pretty straight-forward, but now we need to provide an actual OffsetMapping as well so that Compose can map each character from the original string to the transformed one. Since we're replacing every character with two others, we just need to double (or half) the offset to get to the right result. For example, if we have the string "abcd" mapped to "a.b.c.d.", character "b" in the original string is at index 1 and it maps to index 2 in the output string.

### The credit card field

Armed with all that, let's see how the full example would look like. First, let's extract the visual transformation into a separate function:

```kotlin
            BasicTextField(
                value = cardNumber,
                onValueChange = { cardNumber = it.take(16) },
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
                visualTransformation = { creditCardFilter(it) }
            )
```

We've also added a limitation of 16 characters to the value by only saving the first 16 characters of what the user types.

Then let's add the following:

```kotlin
val mask = "1234  5678  1234  5678"

fun creditCardFilter(text: AnnotatedString): TransformedText {
    val trimmed = if (text.text.length >= 16) text.text.substring(0..15) else text.text

    val annotatedString = AnnotatedString.Builder().run {
        for (i in trimmed.indices) {
            append(trimmed[i])
            if (i % 4 == 3 && i != 15) {
                append("  ")
            }
        }
        pushStyle(SpanStyle(color = Color.LightGray))
        append(mask.takeLast(mask.length - length))
        toAnnotatedString()
    }

    val creditCardOffsetTranslator = object : OffsetMapping {
        override fun originalToTransformed(offset: Int): Int {
            if (offset <= 3) return offset
            if (offset <= 7) return offset + 2
            if (offset <= 11) return offset + 4
            if (offset <= 16) return offset + 6
            return 22
        }

        override fun transformedToOriginal(offset: Int): Int {
            if (offset <= 4) return offset
            if (offset <= 9) return offset - 2
            if (offset <= 14) return offset - 4
            if (offset <= 19) return offset - 6
            return 16
        }
    }

    return TransformedText(annotatedString, creditCardOffsetTranslator)
}
```

We put the desired hint text into a mask constant. These are the digits we're going to display for the not-yet-filled digits of the card number.

The first part of the function deals with transforming the string. We go through the input, adding spaces where needed. We then switch over to Light Gray and pad the string up to 22 characters with digits from the mask.

The last part is the custom offset mapping. While a bit wordy, all it does is that it subtracts (or adds, as required) the number of white spaces we've added to the string.

And that's all there is to it! Compose does the rest. Remember you can browse the full code [here](https://github.com/acristescu/ComposeVisualTransformationStudy). Happy coding!