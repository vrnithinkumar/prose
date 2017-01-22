{% capture pdf %} <a href="{{ include.pdf }}" aria-label="PDF" alt="PDF" title="PDF"><i class="fa fa-file-pdf-o"></i></a> {% endcapture %}
{% capture slides %} {% if include.slides %}<a href="{{ include.slides }}" aria-label="Slides" title="Slides" alt="Slides"><i class="fa fa-file-powerpoint-o"></i></a>{% endif %} {% endcapture %}
### **{{ include.title }}**, <small>{{ include.venue }}</small> &nbsp; {{ pdf }} &nbsp; {{ slides }}
{: .mb-1}
*{{ include.authors }}*
