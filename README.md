# react-commentbox
A generic comment box made in React

## Installation
```bash
npm install react-commentbox --save
```

## Usage
This component has the following default props:
```js
const defaultProps = {
    classPrefix: 'cb-', // prefixes every css class with this
    className: 'commentbox', // the class name of the top-level element
    disabled: true, // disabled when there's no user/author name present
    usersHaveAvatars: false, // if true, your comments must have a userAvatarUrl field 
    levelPadding: 25, // how much to indent each nested comment
    textareaRows: 7, // number of rows initially visible in the textarea
    loadingContent: 'Loading...', // can be a string or instance of a component
    expandButtonContent: '[+]', // can be a string or instance of a component
    contractButtonContent: '[-]', // can be a string or instance of a component
    showReplyButtonContent: 'reply', // can be a string or instance of a component
    hideReplyButtonContent: 'cancel', // can be a string or instance of a component
    postReplyButtonContent: 'Post Reply', // can be a string or instance of a component
    postCommentButtonContent: 'Post Comment', // can be a string or instance of a component
    flagButtonContent: 'flag', // can be a string or instance of a component
    flagButtonDisabledContent: '(flagged)', // can be a string or instance of a component
    postButtonExtraContent: null, // placed next to the comment and reply buttons, can be a string or instance of a component
    disabledComponent: React.Component, // what to show when the user isn't logged in or there's no author info to use
    normalizeComment: (comment) => { // turns your comment object into an object the component expects
        return {
            id, // a unique id for this comment. used as the "key" prop
            flagged, // is this comment flagged for moderation?
            bodyDisplay, // a string or component to be displayed as the comment body
            userNameDisplay, // a string or component to be displayed as the user name
            timestampDisplay, // a string or component to be displayed as the timestamp
            belongsToAuthor, // does this comment belong to the currently logged in author?
            parentCommentId // if this comment is a reply to another comment, set it's id here           
        };
    },
    getComments: () => new Promise(), // replace with a function call that fetches comments in oldest to newest order
    comment: (body) => new Promise(), // replace with a function call that creates a comment
    reply: (body, commentId) => new Promise(), // replace with a function call that creates a reply to a comment
    flag: (commentId) => new Promise() // replace with a function call to flag a comment
};
```

- The `getComments` function should resolve to an array of comments. Each element in that array is run through the supplied
`normalizeComment` function. Comments must be in order of oldest to newest in order to properly determine comment nesting.
- the `comment` function should be replaced with your ajax call to create the comment. It doesn't need to resolve to anything.
- the `reply` function is similar to the `comment` function, except it gives you the `commentId` of the comment the reply is for.
- the `flag` function should be another ajax call to flag that comment.
- comments eventually need an author to attribute themselves to. You manage that on your own via the `disabled` and `disabledComponent` props.

## Example
The below example is an "anonymous" comment implementation, where you just need to type in an author name at the time of posting.
You'll usually want to replace that logic with a login implementation.

This example assumes your comments are coming from Contentful, thus the `normalizeComment` is catered to that use case.

It also assumes that the `/create-comment` and `/flag-comment` endpoints exist. You can find a full, working example
of how to implement these endpoints in Express.js in the `commentbox/api` directory.

```jsx harmony
import React from 'react';
import CommentBox from 'react-commentbox';

class App extends React.Component {

    constructor(props) {
        super(props);
        this.state = {
            authorName: '',
            authorNameIsSet: false
        };
        this.onChangeAuthorName = this.onChangeAuthorName.bind(this);
        this.onSubmitAuthorName = this.onSubmitAuthorName.bind(this);
        this.getComments = this.getComments.bind(this);
        this.comment = this.comment.bind(this);
        this.reply = this.reply.bind(this);
        this.flag = this.flag.bind(this);
        this.disabledComponent = this.disabledComponent.bind(this);
    }

    onChangeAuthorName(e) {

        this.setState({
            authorName: e.currentTarget.value
        });
    }

    onSubmitAuthorName(e) {

        e.preventDefault();

        this.setState({
            authorNameIsSet: true
        });
    }

    getComments() {

        return this.props.contentfulClient.getEntries({
            'order': 'sys.createdAt',
            'content_type': 'comment',
            'fields.blogPost.sys.id': this.props.blogPostId,
        }).then((response) => {

            return response.items;

        }).catch(console.error);
    }

    normalizeComment(comment) {

        const { id, createdAt } = comment.sys;
        const { body, author, parentComment, flagged } = comment.fields;

        return {
            id,
            flagged,
            bodyDisplay: body,
            userNameDisplay: author,
            timestampDisplay: createdAt.split('T')[0],
            belongsToAuthor: false,
            parentCommentId: parentComment ? parentComment.sys.id : null
        };
    }

    comment(body) {

        return this.props.postData('http://localhost:1337/create-comment', {
            body,
            authorName: this.state.authorName,
            blogPostId: this.props.blogPostId
        });
    }

    reply(body, commentId) {

        return this.props.postData('http://localhost:1337/create-comment', {
            body,
            authorName: this.state.authorName,
            blogPostId: this.props.blogPostId,
            parentCommentId: commentId
        });
    }

    flag(commentId) {

        return this.props.postData('http://localhost:1337/flag-comment', {
            commentId: commentId
        });
    }

    disabledComponent() {

        return (
            <form onSubmit={this.onSubmitAuthorName}>
                <input
                    type="text"
                    placeholder="Enter your name to post a comment"
                    value={this.state.authorName}
                    onChange={this.onChangeAuthorName}
                />
                <button type="submit">Submit</button>
            </form>
        );
    }

    render() {

        return (
            <div>
                <h2>
                    Comments
                </h2>
                <CommentBox
                    usersHaveAvatars={false}
                    disabled={!this.state.authorNameIsSet}
                    getComments={this.getComments}
                    normalizeComment={this.normalizeComment}
                    comment={this.comment}
                    reply={this.reply}
                    flag={this.flag}
                    disabledComponent={this.disabledComponent}
                />
            </div>
        );
    };
}
```
