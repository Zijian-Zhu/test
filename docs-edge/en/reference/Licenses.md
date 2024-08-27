---
layout: global
title: Third-Party Licenses
---

## Alluxio
<table class="table table-striped">
<tr><th>Project</th><th>Version</th><th>License</th><th>Source</th></tr>
{% for item in site.data.generated.third-party-licenses %}
  <tr>
    <td><a class="anchor" name="{{ item.project }}"></a> {{ item.project }}</td>
    <td>{{ item.version }}</td>
    <td>{{ item.license }}</td>
    <td>{{ item.source }}</td>
  </tr>
{% endfor %}
</table>
