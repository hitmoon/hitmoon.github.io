---
layout: null
permalink: /
---
<!DOCTYPE html>
<html>
<head>
  <script>
    // 根据浏览器语言重定向
    var userLang = navigator.language || navigator.userLanguage;
    var supportedLangs = {{ site.languages | jsonify }};
    var defaultLang = '{{ site.default_lang }}';
    
    // 检查前两个字符
    var lang = userLang.substr(0, 2);
    
    if (supportedLangs.includes(lang)) {
      window.location.href = '{{ site.baseurl }}/' + lang + '/';
    } else {
      window.location.href = '{{ site.baseurl }}/' + defaultLang + '/';
    }
  </script>
  <meta http-equiv="refresh" content="0; url={{ site.baseurl }}/{{ site.default_lang }}/">
</head>
<body>
  <p>Redirecting to your preferred language...</p>
</body>
</html>
