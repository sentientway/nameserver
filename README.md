# nameserver
## Abstract
This module impliments a distributed nameserver app that performs the function of DNS in the client server model. The Proof of Stake (PoS) model implimented by Cosmos requires a nameserver app for a public blockchain. This requirement will be discussed at greater length in the introduction.

The Cosmos SDK includes this module as a boilerplate to aid in customizing a nameserver module to meet the requirements for the blockchain or *distributed* application. Cosmos offers a tutorial for creating the same nameserver boilerplate using the Cosmos scaffold tool, which is an excellent way to become familar with Cosmos SDK.

This document presents the components within a nameserver application, how the different modules work together in the blockchain application, and which modules to customize to meet application requirements. The goal is to minimize the coding required by the application developer.
## Contents
