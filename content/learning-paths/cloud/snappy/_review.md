---
# ================================================================================
#       Edit
# ================================================================================

# Always 3 questions. Should try to test the reader's knowledge, and reinforce the key points you want them to remember.
    # question:         A one sentance question
    # answers:          The correct answers (from 2-4 answer options only). Should be surrounded by quotes.
    # correct_answer:   An integer indicating what answer is correct (index starts from 0)
    # explination:      A short (1-3 sentance) explination of why the correct answer is correct. Can add aditional context if desired


review:
    - questions:
        question: >
            Lzbench can only be used to measure the performance of snappy and not zstd?
        answers:
            - "True"
            - "False"
        correct_answer: 2                  
        explination: >
            You can use lzbench to measure either snappy or zstd performance

    - questions:
        question: >
            Both zstd and snappy can run on Arm servers?
        answers:
            - "Yes"
            - "No"
        correct_answer: 1                     
        explination: >
            The answer is Yes: Both the compression libraries run on Arm servers
               

# ================================================================================
#       FIXED, DO NOT MODIFY
# ================================================================================
title: "Review"                 # Always the same title
weight: 20                      # Set to always be larger than the content in this path
layout: "learningpathall"       # All files under learning paths have this same wrapper
---
