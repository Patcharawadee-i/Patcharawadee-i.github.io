# Thai posts

Rules for everything under `/th/`. They add to the root `CLAUDE.md`; they do not
replace it.

## No first-person pronoun for the author

- Never use `ผม`.
- Write without a subject wherever Thai allows it, which is almost everywhere:
  - not `ผมอยากสั่ง agent ให้...` but `อยากสั่ง agent ให้...`
  - not `กฎที่ผมตั้งไว้` but `กฎที่ตั้งไว้`
  - not `เครื่องผม` but `เครื่องนี้`
- If a sentence truly cannot stand without a pronoun, use the neutral `ฉัน`.
  Try rewriting the sentence first.
- This applies to code comments in Thai as well.

## Write Thai, not translated English

The Thai version must read smoothly, as if it had been written in Thai first.
The author has asked for this explicitly: smooth Thai matters more than staying
close to the English sentence.

A sentence that keeps English word order and English metaphors is unreadable in
Thai even when every word is correct.

- English figures of speech do not survive translation. Say what they mean:
  - not `คำสั่งใน skill เป็นแค่มารยาท` but `ข้อความใน skill เป็นเพียงคำแนะนำ agent จะไม่ทำตามก็ได้`
  - not `การเดินสายใช้งานได้` but `การเชื่อมต่อใช้งานได้`
  - not `โมเดลทำตัวดี แล้วก็ถูก kill` but `โมเดลรายงานตรงตามจริง แต่ pod ถูก kill เพราะ memory ไม่พอ`
- Headings state the topic plainly. A reader skimming the table of contents
  should know what each section is about.
- When a paragraph explains an event, tell it in the order it happened.

- Break a long English sentence into two or three short Thai ones.
- Say the plain thing. Not `รายงานช่างคุยที่ยกสิ่งที่อ่านมาใส่ จะพาเนื้อหาเอกสารไปอยู่ครบทั้งสามที่`
  but `ถ้ารายงานคัดข้อความจากเอกสารมาใส่ ข้อความนั้นก็จะไปค้างอยู่ทั้งสามที่`.
- Drop English idioms that have no Thai equivalent instead of translating them
  word for word.
- Use the same Thai word for the same thing through the whole post.

## Check before showing a Thai draft

1. Search the file for `ผม`. There must be none.
2. Search for `ฉัน`. Each one needs a reason; rewrite if the sentence works
   without it.
3. Read every paragraph as a Thai reader who has not seen the English version.
   Rewrite any sentence that needs the English to be understood.
