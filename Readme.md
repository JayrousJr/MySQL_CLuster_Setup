# MySQL Cluster Lab

This repository documents how to set up a **MySQL Cluster**: a group of database servers that work together as one system.

## What is it?

A normal database lives on a single server. If that server breaks, the data and the application go down with it. A MySQL Cluster spreads the same data across several servers, so if one fails, the others keep working without losing anything.

## What this repo contains

- A step-by-step guide to building a small cluster: one server that manages the group and three servers that store the data
- The configuration files and commands used to set it up
- Instructions for testing that data written on one server appears on the others
- Notes on backups, restarting after a power cut, and fixing common problems

## Who is it for?

Anyone learning how database clustering works, or planning to set one up on their own servers. It started as a learning lab and is written so it can be repeated on physical servers.

## Why this setup?

- **Reliability:** the database keeps running if a server goes offline
- **Consistency:** every server shows the same, up-to-date data
- **Learning:** a small, hands-on example that is easy to follow and break on purpose

## Getting started

Open the main guide in this repository and follow it step by step.
