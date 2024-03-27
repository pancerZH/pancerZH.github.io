---
sort: 10
---

#  Cautionary Tale of Data Misinterpretation

## Background

This work is related to [CBOR](https://cbor.io/), a specific data format. We received some responses with a special field, which can contain CBOR content or plain content, but we don't always know beforehand if a specific field holds valid CBOR or not. The solution, at first glance, seems reasonable: attempt to parse the field as CBOR. If successful, great! We have our data. If it fails, we know it's not CBOR and move on.

## Where the Disaster Begins

We have some responses with some unexpected content – not encoded in CBOR, but with some bytes that just so happen to mimic the structure of a CBOR `ByteArray`. By recognizing these bytes as `ByteArray`, the parser would allocate some space for these fake CBOR data, and here is the problem: CBOR's `ByteArray` can be arbitrarily large, and our unsuspecting parser allocates a massive chunk of memory to store this phantom data.

We actually received multiple pieces of such content. With each attempt at parsing, a vast amount of memory gets devoured. This memory hogging eventually leads to a system meltdown, causing crashes for other applications as well. We triggered the alert system and restarted the system to temporarily resolve the issue, but some more effective solutions must be taken.

## Lessons Learned

This incident serves as a stark reminder of the perils of polymorphic data and the importance of robust data validation. Here are some key takeaways:

- **Validate, Validate, Validate:** Don't blindly trust incoming data. Implement strong validation checks to ensure data conforms to the expected format before processing.
- **Consider Alternatives:** Perhaps a different approach, like checking for specific markers or patterns within the data, could help identify CBOR content without relying on parsing attempts.
- **Be Wary of Hidden Costs:** When dealing with potentially large data structures like `ByteArray`, have safeguards in place to prevent excessive memory allocation.

