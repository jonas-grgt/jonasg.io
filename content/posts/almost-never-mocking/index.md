
After reading the article I believe it contains a fatal flaw, namely testing implementation details. Because that is what you are doing if you are focused on testing classes and code. You do not test the code you test the behavior and through doing that you test the code. So you need to look for the API (as in interface or gate) that leads to that behavior. Within the refactoring step that interface should never change unless you misinterpreted the behavior you are implementing. Hence this allows you to move things around as freely as possible during the refactoring phase while not breaking your tests.

If you are testing classes you are often forced to use mocks and it will not allow you to refactor freely.

I do agree with the sentiment that TDD is over evangelized, it is also very difficult and takes a lot of knowledge in general, like stated in the article, on software design and patterns.

```java

    @Test
    @WithMockUser
    void linkBudgetWithBmatPlaylistShouldReturn202() throws Exception {
        // given
        var budgetId = UUID.randomUUID();
        var playlistLinkDTO = new PlaylistLinkDTO(PlaylistSource.BMAT, List.of(new LinkablePlaylist(1, "df"), new LinkablePlaylist(2,"df")));

        // when
        this.mvc.perform(post("/budgets/{budget-id}/link", budgetId).with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(playlistLinkDTO)))
                .andExpect(status().isAccepted());

        // then
        Mockito.verify(b1ntvService).linkB1ntvWithBmatPlaylists(Mockito.eq(budgetId), Mockito.argThat(actual -> actual.containsAll(playlistLinkDTO.getLinkedPlaylists())));
    }
    


```

Can't move on unless.I update the service because it has changed 
